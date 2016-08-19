# Routing Data Structure Changes
The underlying data structure that a router template can use has changed in origin release 1.3
This document explains the changes and possible actions needed if someone is upgrading from origin 1.2 to origin 1.3! Anyone attempting to modify the pre-packaged template will also find the information useful.


###Short summary of changes

In the older model, the top level had one map of all services, and to get to routes of the system, one has to iterate over all the services and then get to the routes that each service holds.
In the new model, the top level contains two maps - one for all the routes and one for all the services. Get to any of them now without double iteration.


###The new model explained

The new data structure defining the routing backends consists of two structures representing services and routes, and one top level structure that contains a map to both.

ServiceUnit <-> Service
ServiceAliasConfig <-> Route

Top level router template has two maps:
```
State            map[string]ServiceAliasConfig
ServiceUnits     map[string]ServiceUnit
```

In the older model, each route had only one service. So while a route could hold only one service, a service could be part of many routes. In the new model, route can have many services and any service can be part of many routes. This required a change of data structures. The ServiceAliasConfig now holds a map of ServiceUnitNames with their corresponding weights. To get to the actual 'service'/'ServiceUnit', one needs to look up the top level map 'ServiceUnits'

```
type ServiceAliasConfig {
  ..
  ..
  ServiceUnitNames map[string]int32
}
```

Lets quickly go through all the routes as an example:

1. Iterate over template.State map (gives us all the routes, represented by ServiceAliasConfig)
2. Go over all services of a route along with their weights.
3. With each service name, look up the actual service from the template.ServiceUnits map
4. Go over endpoints of the service with the Endpoints field in ServiceUnit structure and use those endpoints with the associated weight for the service.




###The older model explained

A service could be part of many routes, so we had two basic structures still - 
ServiceAliasConfig <-> Route
ServiceUnit <- which maps a Service, but also has one extra field that contains all the ServiceAliasConfigs(routes) that it is part of.

```
type ServiceUnit {
 ..
 ..
  ServiceAliasConfigs map[string]ServiceAliasConfig
}
```

The top level template just had a map of all Services in the system. To iterate to routes, one would just have to iterate over 'services' first and get to the routes that it is part of. i.e.
1. Iterate over all ServiceUnits (services)
2. Iterate over all ServiceAliasConfigs(Routes) that this Service has
3. Get the route info (header/tls etc) and use the ServiceUnit's Endpoints field to get to the actual backends.


