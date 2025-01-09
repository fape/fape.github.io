---
layout: post
title: Logoláshoz OpenAPITools-szal egyedi Operation/api generálás Node.js esetén
---

# Probléma
Egy Node.js-es, azonbelül is NestJS-s projektben szerettem volna a backend hívások logolását kiegészíteni a "logikai" végpont infókkal. Könnyebb legyen aggregálni és tendenciákat vizsgálni.
Pl a swagger/openapi-s példa Pet store esetén, ha a hívás `GET https://petstore3.swagger.io/api/v3/pet/123` akkor a logban `GET api/v3/pet/{id}` (vagy hasonló) kerüljön. 

# Előfeltételek
* [@openapitools/openapi-generator-cli](https://www.npmjs.com/package/@openapitools/openapi-generator-cli) van használva.
  * `typescript-nestjs` generátorral
  * DE egyedi `Mustache` templatekkel
  
# Ötlet #1
Generáláskor rakjunk plusz infókat a `service`-be.

A `path` már most is használva van a template fájlban. Amiről kiderül hogy a [`TypeScriptNestjsClientCodegen.java`](https://github.com/OpenAPITools/openapi-generator/blob/d30220b8dfe60dacbdf01e865150765e6ae6e431/modules/openapi-generator/src/main/java/org/openapitools/codegen/languages/TypeScriptNestjsClientCodegen.java#L297)-ból, hogy felül van írva az eredeti interface leíróhoz képest.

Azaz `GET api/v3/pet/${encodeURIComponent(String(id))}` lesz.

# Ötlet #2
Milyen más template változók vannak? 
A [`CodegenOperation.java`](
https://github.com/OpenAPITools/openapi-generator/blob/d30220b8dfe60dacbdf01e865150765e6ae6e431/modules/openapi-generator/src/main/java/org/openapitools/codegen/CodegenOperation.java#L25)-ból kiderült pár igéretes. Pl `operationId` (régi nevén `nickname`, ez is volt már használva template-ben) valamint `operationIdCamelCase`.

Az [openapi-generator Debugging Templates](https://github.com/OpenAPITools/openapi-generator/blob/master/docs/debugging.md#templates) doksi szerint lehet a  `JAVA_OPTS` környezeti változóval több infót kinyerni. pl `export JAVA_OPTS="--global-property debugOperations"`. Csak erre hiba jött:

```shell
Error: Unrecognized option: --global-property
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
```

Végül az api szintű `config.yaml`-be került:
(A doksi nem említi, vagy nem jött szembe, de egy pull request](https://github.com/OpenAPITools/openapi-generator/pull/8339) igen)

```yaml
globalProperties:
    debugModels: true
    debugOperations: true
```

Ennek hatására kaptam egy 38 000 soros json-t, amiből látszott, hogy miből lehet választani a templateben.

# Megoldás
Végül kombinálva a két ötletet, ez került az `api.service.mustache` template fájlban:

<!-- {% raw %} -->
```mustache
metadata: { 
    operationId: '{{{moduleClassName}}}.{{{operationIdCamelCase}}}', 
    operationName: '{{{httpMethod}}} {{{contextPath}}}{{{path}}}'
}
```
<!-- {% endraw %} -->

