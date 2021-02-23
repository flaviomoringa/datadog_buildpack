# Datadog Decorator

Work in progress.

Yep! I lied this is not a full fledged buildpack but a decorator. 

This is a Cloud Foundry decorator, unlike a "real" buildpack, a decorator doesn't produce a droplet from source. Instead, it "decorates" (hence the name) an already produced droplet to implement some kind of add-on feature.

This decorator injects Datadog environment variables inside the droplet.

For more information on the available environment variables: 

https://docs.datadoghq.com/tracing/setup_overview/setup/java/?tab=otherenvironments#configuration


## Deploying

```
cf push -f manifest-alpha.yml \ 
  -b https://github.com/flaviomoringa/datadog_buildpack.git  \
  -b datadog_application_monitoring \
  -b java_buildpack
```

# How it works


There's three sources for Datadog environment variables:

1. The manifest file
2. A user provided service tagged with "datadog"
3. The buildpack's defaults

Except for special cases, they're applied in that order, meaning that the manifest file can override the service values and defaults are only used as last resource.

In general, however, is cleaner to keep all Datadog configurations in the user provided service to avoid cluttering the manifest with too manyy variables and to keep everything related to Datadog in the same place.


## Special cases

The `DD_API_KEY` can't be set in the manifest and must be in the service.

`JAVA_OPTS` must be set in the manifest. If it does not include a Datadog agent (named `dd-java-agent-[0-9.]*.jar`) as a javagent and some is found in the filesystem, that one is added to `JAVA_OPTS`.

`DD_TAGS` defined in the Datadog service are merged with the defaults, the former overriding the latter on key conflicts. However, defining `DD_TAGS` in the manifest allows to override it as a whole, meaning no values from the service or defaults are considered.

`DD_VERSION` can be explicitly set in the manifest or otherwise can be infered from the `Implementation-Version` key in `META-INF/MANIFEST.MF`, if present.


# How to use it

## Create a service

Create a PCF user provided service containing the relevant variables, eg:

```bash
cf cups appx-datadog -t "datadog" -p '{"DD_API_KEY":"53cr37"}'
```

The important here is adding the tag "datadog" to the service, you then can edit the content of the service in the AppsManager of PCF, if you are more visually inclined. 

## Update the manifest


The manifest must include the datadog service and the buildpacks in this order:

```yaml
applications:
  - name: appx
    buildpacks:
      - dh_io_datadog
      - datadog_application_monitoring
      - java_buildpack
    ...
    services:
      - appx-datadog
      - ...
```
Since the `dh_io_datadog` buildpack is not published inside PCF buildpack manager you can replace it with the repository url: https://github.com/lmmendes/datadog_buildpack.git 

# Default env values

The decorator will expose the following variables

| Variable  | Default  | Observation  |
|---|---|---|
| DD_API_KEY | None | Needs to be declared inside the User Defined Service from PCF |
| DD_ENV| Inferred from the PCF space | |
| DD_SERVICE | Application name inferred from PCF app name |  | 
| DD_SITE | datadoghq.eu | |
|RUN_AGENT | true ||
| DD_APM_ENABLED | true ||
| DD_LOGS_ENABLED | true | |
| DD_LOGS_INJECTION | true ||
| DD_TRACE_ANALYTICS_ENABLED | true ||
| DD_ENABLE_CHECKS | false ||
| STD_LOG_COLLECTION_PORT | 10514 ||
|DD_PROPAGATION_STYLE_INJECT| Datadog,B3 ||
|LOGS_CONFIG | type: tcp, port: STD_LOG_COLLECTION_PORT, source: pcf, service: DD_SERVICE |
| DD_TAGS | service: inferred from pcf app name , tenant: inferred from pcf organization name  , version: retrived from the  |
