# Routing Data Structure Changes
The underlying data structure that a router template can use has changed in origin release 1.3  
Extra action may be needed for an upgrade from 1.2 to 1.3 release. Please use this document to know how to do the changes to your template.


Contents:
 - [Short summary of changes](#short-summary-of-changes)
 - [The new model](#the-new-model-explained)
 - [The older model](#the-older-model-explained)
 - [I want to upgrade](#upgrade-actions)

###Short summary of changes

In the older model, the top level has one map of all services, and to get to routes of the system, one has to iterate over all the services and then get to the routes that each service holds.  
In the new model, the top level contains two maps - one for all the routes and one for all the services. Get to any of them now without double iteration.


###The new model explained

The new data structure defining the routing backends consists of two structures representing services and routes, and one top level structure that contains a map to both.
```
ServiceUnit <-> Service
ServiceAliasConfig <-> Route
```

Top level router template has two maps:
```
State            map[string]ServiceAliasConfig
ServiceUnits     map[string]ServiceUnit
```

In version1.3+, a Route can have many services and any service can be part of many routes. The ServiceAliasConfig(Route) holds a map of ServiceUnitNames(Service) with their corresponding weights. To get to the actual 'service'/'ServiceUnit', one needs to look up the top level map 'ServiceUnits'

```
type ServiceAliasConfig {
  ..
  ..
  ServiceUnitNames map[string]int32
}
```

Lets quickly go through all the routes as an example:

1. Iterate over template.State map (gives us all the routes, represented by ServiceAliasConfig)
2. Go over all services of a route along with their weights
3. With each service name, look up the actual service from the template.ServiceUnits map
4. Go over endpoints of the service with the Endpoints field in ServiceUnit structure and use those endpoints with the associated weight for the service.
Example code:
```
# get the routes/ServiceAliasConfigs from .State
{{ range $routeId, $route := .State }}
  # get the names of all services that this route has, with the corresponding weights
  {{ range $serviceName, $weight := $route.ServiceUnitNames }}
    # now look up the top level structure .ServiceUnits to get the actual service object
    {{ with $service := index $.ServiceUnits $serviceName }}
      # get endpoints from the service object
      {{ range $idx, $endpoint := endpointsForAlias $route $service }}
# print the endpoint
server {{$endpoint.IdHash}} {{$endpoint.IP}}:{{$endpoint.Port}}...

```



###The older model explained

A service could be part of many routes, so we have two basic structures - 
```
ServiceAliasConfig <-> corresponds to a Route
ServiceUnit <- corresponds to a Service, but also holds how many Routes point to it
```
ServiceUnit has one special field that contains all the ServiceAliasConfigs(routes) that it is part of:
```
type ServiceUnit {
 ..
 ..
  ServiceAliasConfigs map[string]ServiceAliasConfig
}
```

The top level template just has a map of all Services in the system. To iterate to routes, one would just have to iterate over 'services' first and get to the routes that it is part of. i.e.
1. Iterate over all ServiceUnits (services)
2. Iterate over all ServiceAliasConfigs(Routes) that this Service has
3. Get the route info (header/tls etc) and use the ServiceUnit's Endpoints field to get to the actual backends.

Example code:
```
{{ range $id, $serviceUnit := .State }}
  {{ range $routeId, $route := $serviceUnit.ServiceAliasConfigs }}
    {{ range $idx, $endpoint := endpointsForAlias $route $serviceUnit }}
server {{$endpoint.IdHash}} {{$endpoint.IP}}:{{$endpoint.Port}}
```

But with the older model we cannot accommodate the idea that a route can contain multiple services.

###Upgrade Actions
If you are upgrading from version 1.2 to version 1.3 of openshift origin but you never changed the default haproxy routing template that came with the image, then nothing much to do. Just ensure that the new router image is used so that you can use the latest features of the release. Come back to this document if you ever need to change the template.   
  
If you ever customized your haproxy routing template then, depending on the changes, you may want to
  - re apply the changes on the newer template. Or,
  - rewrite your existing template using the newer model
    * Iterating over .State now gives ServiceAliasConfigs and not the ServiceUnits
    * Each ServiceAliasConfig now has multiple ServiceUnits in it stored as keys of a map, where the value of each key is the weight associated with the service
    * To get the actual service object, index over another top level object called 'ServiceUnits'
    * One cannot directly get the list of routes that a service serves to now. We have found no use of that information. If you use this information for any reason, you will have to construct your own map by iterating over all routes that contain a particular service.  
   
It is recommended that the new template is taken as a base and the changes/modifications are re-applied on it. Then, rebuild the router image. Same applies if you use configMap to supply the template to the router - you will have to use the new image or rebuild your image anyway because the openshift executable inside the image needs an upgrade too.
