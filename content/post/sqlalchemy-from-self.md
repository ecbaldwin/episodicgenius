+++
Tags = ["Openstack", "Neutron", "Development", "SqlAlchemy"]
date = "2016-09-10T09:23:55-06:00"
title = "Experience with sqlalchemy from_self"

+++

I was playing around with [a change][change] yesterday hoping to
optimize out an extra database round trip. I thought I could accomplish
it without having to change the structure of the code at all just by
replacing a list comprehension -- which triggered a query to the
database -- with just a query which could be passed directly to the next
query. This would save a round-trip and should be less racy.

The idea was to replace this...

    return [port.id for port in query]

... with this...

    return query.from_self(models_v2.Port.id)

The result of either expression could be passed in to the sqlalchemy
[notin\_  operator][notin] with essentially the same results but the
latter would trigger one less conversation with the DB. The idea seemed
sound and in fact it was.

I noticed what I thought was a strange anomaly. I ran [a few of the
existing unit tests][unittests] with only a slight change to generate a
list from the query expression. This first test failed because the query
now produced two rows instead of one! I couldn't figure out why using
from_self to change the result columns would change the number of
results.

I pulled it up in the debugger.

    $ .tox/py27/bin/python -m testtools.run neutron.tests.unit.plugins.ml2.drivers.l2pop.test_db.TestL2PopulationDBTestCase.test__get_ha_router_interface_ids_with_ha_dvr_snat_port
    Tests running...
    > /home/carl/Openstack/neutron/neutron/plugins/ml2/drivers/l2pop/db.py(91)_get_ha_router_interface_ids()
    -> return query.from_self(models_v2.Port.id).distinct()
    (Pdb) len([p for p in query])
    1
    (Pdb) len([p for p in query.from_self(models_v2.Port.id)])
    2

I quickly found that the two results were really just the same port id
duplicated. This was a relief because when I pass this in to the notin\_
operator, it will function equivalently. I could get [the unit
tests][unittests] to pass easily by adding ".distinct()" to the query.

    (Pdb) [p for p in query.from_self(models_v2.Port.id)]
    [(u'1c8a0a09-c605-4137-aad6-d45b4b2b9f65',), (u'1c8a0a09-c605-4137-aad6-d45b4b2b9f65',)]
    (Pdb) len([p for p in query.from_self(models_v2.Port.id).distinct()])
    1

I had the issue solved but my curious mind wouldn't leave it alone.

From reading [the docs][fromself], I expected that the [new query
produced after applying from_self][newquery] would just be a new select
wrapped around the [old query][origquery]. To my astonishment, the [new
query][newquery] was *a lot* different than [the original][origquery].
It was much simpler. A lot of the [original query][origquery] was
optimized out. All of the result columns from all of the various tables
that I wouldn't end up reading were gone. Even more, all of the left
outer joins to tables that I didn't need to look at were gone with them.
This all made sense to me and I was glad to see it; I was quite
impressed even.

I kind of want to use from_self a lot more in code after seeing this.
For one thing, it produces SQL that is much easier to understand when I
need to debug code involving a query. For another, the simplified query
must perform a little better because it doesn't have to select results
from all of the left outer joins or even consider any of those tables at
all.

But still, why the duplicatated result in the new query? At first glance
it didn't make any sense to me and it disturbed me a little that the
optimizations would end up selecting a different number of rows.

So, I set my brain loose on the problem. I started by looking over the
python code to construct the query.

    query = session.query(Port)
    query = query.join(L3HARouterAgentPortBinding,
        L3HARouterAgentPortBinding.router_id == Port.device_id)
    return query.filter(
        Port.network_id == network_id,
        Port.device_owner.in_(HA_ROUTER_PORTS))

It didn't take me long to convince myself that I *should* expect two
rows from this query in either case. In this case, there is one port
that matches the filters and there are two rows from the binding table
that match the join condition from that port. Now the question is why
does the more complex query end up squashing the duplicate result?

I reran the unit test so that I could [get it to write a file based
sqlite database][accesssqlite]. I ran the [original query][origquery]
and it produced two results!

    sqlite> SELECT ports.project_id AS ports_project_id,
    (snip)
       ...> FROM ports JOIN ha_router_agent_port_bindings ON ha_router_agent_port_bindings.router_id = ports.device_id
    (snip)
       ...>     LEFT OUTER JOIN ipallocations AS ipallocations_1 ON ports.id = ipallocations_1.port_id
    (snip)
       ...> WHERE ports.network_id = 'network_id' AND ports.device_owner IN ('network:ha_router_replicated_interface', 'network:router_centralized_snat');
    |e40977eb-63cf-4f39-9b76-e2c4a94f8a9e||network_id|fa:16:3e:b4:a4:00|1|ACTIVE|router_id|network:router_centralized_snat|6|||||2016-09-13 21:53:09.531416||6|ports||1||||||||||||||||||||||||||||||||||||||||||||||||||||e40977eb-63cf-4f39-9b76-e2c4a94f8a9e|localhost|normal||unbound|||||
    |e40977eb-63cf-4f39-9b76-e2c4a94f8a9e||network_id|fa:16:3e:b4:a4:00|1|ACTIVE|router_id|network:router_centralized_snat|6|||||2016-09-13 21:53:09.531416||6|ports||1||||||||||||||||||||||||||||||||||||||||||||||||||||e40977eb-63cf-4f39-9b76-e2c4a94f8a9e|localhost|normal||unbound|||||

So did the new query.

    sqlite> SELECT anon_1.ports_id AS anon_1_ports_id
       ...> FROM (SELECT ports.project_id AS ports_project_id,
       ...>           ports.id AS ports_id,
    (snip)
       ...>       FROM ports JOIN ha_router_agent_port_bindings ON ha_router_agent_port_bindings.router_id = ports.device_id
       ...>       WHERE ports.network_id = 'network_id' AND ports.device_owner IN ('network:ha_router_replicated_interface', 'network:router_centralized_snat')
       ...>       ) AS anon_1;
    e40977eb-63cf-4f39-9b76-e2c4a94f8a9e
    e40977eb-63cf-4f39-9b76-e2c4a94f8a9e

I'm not sure where to go from here. Something must be collapsing the
results in the first query. Unfortunately, I'm out of time for now. If I
get back to this, I'll post an update.

[change]: https://review.openstack.org/#/c/255237/54..55/neutron/plugins/ml2/drivers/l2pop/db.py@90
[fromself]: http://docs.sqlalchemy.org/en/latest/orm/query.html#sqlalchemy.orm.query.Query.from_self
[notin]: http://docs.sqlalchemy.org/en/latest/core/sqlelement.html#sqlalchemy.sql.expression.ColumnElement.notin_
[origquery]: http://paste.openstack.org/show/570958/
[newquery]: http://paste.openstack.org/show/570959/
[unittests]: https://review.openstack.org/#/c/255237/54..55/neutron/tests/unit/plugins/ml2/drivers/l2pop/test_db.py@188
[accesssqlite]: https://github.com/openstack/neutron/commit/98838c491a5dc8c1d349a175401f80f2a912cc35

<!-- vim:set tw=72 ft=markdown: -->
