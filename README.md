# Table of contents

1. [RAML conventions](#raml-conventions)

    1.1 [Why?](#why)
    
    1.2 [Version](#version)
    
    1.3 [No dependencies to outside resources](#no-dependencies-to-outside-resources)
    
    1.4 [Directory structure](#directory-structure)
    
    1.5 [Specification guidelines](#specification-guidelines)
    
    1.6 [Included resources](#included-resources)
    
    1.7 [Data Types](#data-types)
    
    1.8 [Request and Response bodies](#request-and-response-bodies)
    
    1.9 [Naming conventions](#naming-conventions)
    
    1.10 [Type reusability](#type-reusability)
    
2. [Vision](#vision)
    
    2.1 [Draft structure for automated library generation](#draft-structure-for-automated-library-generation)
    
    2.2 [Convention improvements](#convention-improvements)
    
    2.3 [Naming](#naming)
    
    2.4 [Concept](#concept)

# RAML conventions

## Why?

These conventions are needed to standartize RAML specifications.

Main goal is to have RAML API specification, which can be consumed to generate API Clients or API Servers whithout any custom tweaking.

By writing RAML API we always follow rules defined in [REST](https://github.com/paysera/rest-style-guide) Paysera conventions.

### raml-code-generator

This tool currently allows to generate PHP and JavaScript REST API Clients. It should be availabe in developer host machine by running raml-code-generator.

#### Commands

##### `release:clients`

This command should be used instead of others described below. It makes a lot of automated tasks for you.
Needed arguments:

* path_to_config - path to config.json.

Optional arguments (if not provided, you will be asked in terminal):

* --version_constraint - the constraint in terms of semantic versioning. Used in `package.json` and `git tag`. One of `major`, `minor`, `patch`.
* --commit_message - commit message
* --changelog_entry - entry in `CHANGELOG.md` according to https://keepachangelog.com/en/1.0.0/. Version block should be skipped as it will be resolved from `version_constraint`.

This command will
1. clone repository
1. generate clients from `raml`
1. update changelog
1. show you the difference between current and generated files
1. generate dist files for javascript
1. create a commit
1. push commit
1. create tag

Config format:

```json
{
  "SomeApiName": {
    "raml_file": "path/to/api.raml",
    "clients": {
      "javascript": {
        "repository": "ssh://some.hostname.com/source/js-some-monorepo.git/packages/some-api-client",
        "library_name": "@vendor/some-api-client",
        "client_name": "SomeApiClient"
      },
      "php": {
          "repository": "ssh://some.hostname.com/source/some-api-client-dedicated-repo.git",
          "library_name": "vendor/lib-some-api-client",
          "namespace": "Vendor\\Client\\SomeApiClient"
      }
    }
  }
}
```


##### `php-generator:rest-client`

Used to generate PHP REST API Client. Usage: raml-code-generator php-generator:rest-client path_to_raml output_dir namespace.  Needed arguments:

* path_to_raml - path to main RAML file.
* output_dir - directory where to put generated structure.
* namespace - the namespace of generated classes.

##### `php-generator:symfony-bundle`

Used to generate Symfony Bundle for given raml definition. Usage: raml-code-generator php-generator:symfony-bundle path_to_raml output_dir namespace. Needed arguments:

* path_to_raml - path to main RAML file.
* output_dir - directory where to put generated structure.
* namespace - the namespace for generated Bundle.

Currently generated structure:
* Entities
* Controllers
* Normalizers
* Routing
* Services
* Repositories
* Rest Api configuration
* Result providers
* Entity resolvers

Generated code has some `TODO: generated_code` comments, generally these marks business logic or places where it is impossible to generate something automatically. Result should be used to replace boilerplate when creating Bundle for API.


##### `js-generator:package`

Used to generate PHP REST API Client. Usage: raml-code-generator js-generator:package path_to_raml output_dir client_name.  Needed arguments:

* path_to_raml - path to main RAML file.
* output_dir - directory where to put generated structure.
* client_name - the name of main Client class.

## Version

For each new RAML specification we use RAML 1.0 syntax. Specification details can be found [here](https://github.com/raml-org/raml-spec/blob/master/versions/raml-10/raml-10.md)

## No dependencies to outside resources

There should be no dependencies to resources outside your specification directory.

In example:

It may be tempting to use generic Filter type or trait for SQL-like filtering stored somewhere in directory above. But this creates hidden dependency, implicitly relies on directory structure above, and renders API specification directory to be unportable.

Solution:

Just use Filter from [PayseraCommon RAML library](https://github.com/paysera/lib-raml-common).

## Directory structure

We put *traits*, *types*, *examples* or other structures in their own directories. Main RAML specification file should be named `api.raml`.
Sample directory structure:

```text
{api_name}/
├── api.raml
├── examples
│   └── one-example.json
├── traits
│   ├── other-trait.raml
│   └── some-trait.raml
└── types
    ├── another-type.raml
    └── my-type.raml
```

Where `{api_name}` is the name of the API this RAML specification is created for (e.g. account, transfer, category).

## Specification guidelines

### BaseUri

We use final, publicly available url. We do not use urls for docker environment, localhost or other publicly not available urls.

bad:

```
#%RAML 1.0
title: Client API
version: 1.0
baseUri: http://evpbank.dev.docker/account/rest/v1
```
good:

```
#%RAML 1.0
title: Client API
version: 1.0
baseUri: https://gateway.paysera.com/account/rest/v1
```

### Included resources

We always include all required resources (traits, types, examples). We do not describe them in main api.raml file.

We separate each resource in individual file - this allows more flexible reusability and validation by header line.

bad:

```
types:
  AccountResult:
    displayName: Account Result
    type: object
    properties:
      accounts:
        required: false
        displayName: accounts
        type: array
        items:
          type: Account
```
good:

```
types:
  AccountResult: !include types/account-result.raml
```

### Data Types

#### Fragments

We declare RAML fragment name in header line.

For type we put #%RAML 1.0 DataType as first line in RAML file.

We put each DataType into individual file.

Same applies for other [supported fragments](https://github.com/raml-org/raml-spec/blob/master/versions/raml-10/raml-10.md#typed-fragments)

Fragment declaration helps parsers to recognize and validate specific fragment for available/required fields, etc.

#### Properties

* We always fully describe DataType property
* We avoid duplicate or redundant definitions
* We quote string enums

bad:
```
properties:
  some_field?: string
  other_field: string[]
  third_field:
    displayName: third_field
    description: third_field
  fourth_field:
	type: string
    enum:
      - some value
      - other value
```

good:

```
properties:
  some_field:
    required: false
    type: string
  other_field:
    type: array
	items:
      type: string
  third_field:
    description: Field used for this and that
  fourth_field:
	type: string
    enum:
      - 'some value'
      - 'other value'
```
This helps easily add additional parameters to property without rewriting it. Also some parsers may find it difficult to resolve all shortcuts in property definition. Description looks more standardized and clean.

#### Objects

In DataTypes we use generic object names instead of including object type directly.

bad:
```
properties:
  some_field:
    type: !include: path/to/other-object.raml
```
good:
```
properties:
  some_field:
    type: AnotherObject
```
This allows us to define exact types in root api.raml and have all includes there.

### Request and Response bodies

All resources should have proper bodies declared. We provide example JSON for all bodies.

#### 200 Response
bad:
```
/transfers:
  get:
    displayName: Transfers
    description: Get list of transfers by filter
    responses:
      200:
        description: Success
        body:
          application/json:
            displayName: response
            type: FilteredTransfersResult
```
also bad:

```
/transfers:
  get:
    displayName: Transfers
    description: Get list of transfers by filter
    body:
      application/json:
        displayName: body
        type: any
    responses:
      200:
        description: Success
        body:
          application/json:
            displayName: response
            type: any
```

First example above is missing the request body, second example tells that all bodies (request and response) are of type any - Any is not a specific type and is not allowed.

good:

```
/transfers:
  get:
    displayName: Transfers
    description: Get list of transfers by filter
    body:
      application/json:
        displayName: body
        type: TransfersFilter
    responses:
      200:
        description: Success
        body:
          application/json:
            displayName: response
            type: TransfersResult
            example: !include examples/transfer-result.json
```

Now it is clearly visible what kind of bodies this endpoint consumes and returns.

#### 204 Response

No Content response should not be mixed with 200 response:

```
/transfers:
  delete:
    description: Deletes the transfers
    body:
      application/json:
        displayName: body
        type: TransfersFilter
    responses:
      204:
        description: Success
        body: null
```

## Naming conventions

Naming conventions are needed to help code generator to recognize Entity type.

Fancy types, like *FilteredCategoriesResult* should be avoided.

### Variables

Use of *camelCase* for placeholders in URL is recommended. Do not use dash-separated (kebab-case) for variables, because code generator will fail to convert it to proper variable name.

For property names use *snake_case*

Other parts of URL should follow [REST design conventions](https://github.com/paysera/rest-style-guide#urls):

bad:
```
/{file-id}/get_contents:
  get:
    displayName: Download user attached document
      responses:
        200:
          body:
            binary/octet-stream:
```
good:
```
/{fileId}/get-contents:
  get:
    display_name: Download user attached document
      responses:
        200:
          body:
            binary/octet-stream:
```

### Datetime

In case you want timestamp property to be mapped to DateTime instance, you should add a (datetime_timestamp) annotation to property (type must be integer). In case you have other [supported date types](https://github.com/raml-org/raml-spec/blob/master/versions/raml-10/raml-10.md#built-in-types), you can declare them usual way:

```
timestamp_property:
  (datetime_timestamp):
  type: integer
  description: Some date instance
datetime_property:
  type: datetime-only
  description: Date in RFC3339 format
```

Please read more: https://github.com/raml-org/raml-spec/blob/master/versions/raml-10/raml-10.md#date

### Filters

Filter types are recognized by name match *Filter (i.e. AccountFilter, CategoryFilter, etc.).

### Results

Result types are recognized by name match *Result (i.e. AccountsResult, CategoryResult, etc.).

### Metadata

Medatada types are recognized by name match *Metadata (i.e. ResultMetadata, etc.). Usually

### Entities

All other names are matched as Entity.

In case you must use fancy name, like TransfersBatchResult, you can annotate it to be recognized as Entity by providing custom (entity_type) annotation:

```
  TransfersBatchResult:
    displayName: Result of TransfersBatch
    type: object
    (entity_type):
    properties:
      revoked_transfers:
        type: array
        items:
          type: TransferOutput
      reserved_transfers:
        type: array
        items:
          type: TransferOutput
```

## Inheritance

Single inheritance is supported in generated classes. There are some differences how inheritance is defined in types and traits:

```
types:
  ClientLegal:
    displayName: Legal Client
    type: Client
    properties:
      company_name:
        required: false
        type: string

traits:
  UserFilter:
	is:
	  - Filter
	queryParameters:
	  username:
		type: string
		required: false
```
As you see, in type definition, the parent class is described in type facet; in trait - it is described as traits array in is facet.

Please note that although RAML supports multiple inheritance, we do not use it as currently we do not generate code for language supporting it, so these definitions are invalid:

```
types:
  ClientLegal:
    type: [Client, User]
	properties:
	...

traits:
  UserFilter:
	is:
	  - Filter
	  - Paged
	queryParameters:
	...
```

It is encouraged to use inheritance in RAML definitions as this reflects more natural generated classes structure.

## Type reusability

We avoid copy-pasta pattern in writing RAML. To reuse popular components (types, traits, etc.) we use Libraries. For most of RESTful APIs https://github.com/paysera/lib-raml-common library should suit your needs.

You can follow instructions on how to use RAML libraries in official documentation https://github.com/raml-org/raml-spec/blob/master/versions/raml-10/raml-10.md/#libraries, but this is how you should use it:

* declare uses block in your main api.raml
* name library you want to import i.e. (Paysera) In case library is stored in repository, use concrete version constraint of raw main library file.

```
title: #...
baseUri: #...

uses:
  Paysera: https://raw.githubusercontent.com/paysera/lib-raml-common/x.x.x/rest.raml
types:
  Category: !include types/category.raml
  CategoryResult: !include types/category-result.raml
traits:
  CategoryFilter: !include traits/category-filter.raml

/categories:
  get:
    description: Get categories list
    is:
      - CategoryFilter
    responses:
      200:
        body:
          application/json:
            type: array
            items:
              type: CategoryResult
  post:
    description: Create category
    body:
      application/json:
        displayName: body
        type: Category
```

Now any Result object sould extend Paysera.Result and you only need to override items property, Paysera.Result type already has delfaut _metadata as Paysera.ResultMetadata.

```
# %RAML 1.0 DataType
displayName: Category Result
type: Paysera.Result
properties:
  items:
    required: true
    displayName: Categories
    type: array
    items:
      type: Category
```

Same applies for any Filter object - if it supports standard SQL-style filtering, it should extend Paysera.Filter

```
# %RAML 1.0 Trait
usage: Used to filter Categories
is:
  - Paysera.Filter
queryParameters:
  name:
    required: false
    description: Category name
    type: string
```

### Other reusable types

#### Money

In case you want to use Money objects in your RAML, this type is already defined in Paysera library (link above). RAML generator will map any *.Money type to Money object from evp/money package.

## Custom annotations

Just make sure you add the annotationTypes on root node as described in RAML spec:

```
annotationTypes:
  entity_type: nil
```

### Currently recognized annotations

* entity_type - used to mark type as entity. Useful if type name is mistakenly treated as Filter or Result, i.e. FilteredCategoriesResult.
* datetime_timestamp - used to mark property as timestamp. This allows to properly handle Date/DateTime instances in these properties.

# Vision

## Draft structure for automated library generation

Further described a draft structure needed to automate library generation from raml specification.

## Convention improvements

**No dependencies to outside resources**

There should be no dependencies to resources outside your specification directory with exception to use of common Types/Traits between specifications inside a structure, i.e.: some-resource and another-resource can have common Types/Traits only if they belongs to a logical parent unit (purpose, parent resource, etc.)

## Naming
* library - it is a set of one or multiple raml definitions under the same namespace and for same purpose.
* raml (or raml definition/specification) - set of raml files describing specific resource access and/or manipulation.

## Concept
* api-spec
    * Project should have a configuration file with described clients. Proposed structure below. Each client configuration should contain all necessary data needed to generate and release individual library. Library can contain more than single raml definition. In following scenario separate Client classes should be created.
    
```
{
    "clients": [
        {
            "vendor": "paysera",
            "composer": {
                "name": "paysera/lib-some-resource-client",
                "description": "SomeResource Rest Client"
            },
            "namespace": "Paysera//Client//SomeResourceClient",
            "repository": "ssh://phabricator.dev.lan/diffusion/xxx/lib-some-resource-client.git",
            "raml_definitions": [
                "app-checkout/some-resource",
                "app-checkout/some-other-resource"
            ]
        },
        {
            "vendor": "paysera",
            "composer": {
                "name": "paysera/lib-another-resource-client"
            },
            "namespace": "Paysera//Client//AnotherResourceClient",
            "repository": "https://github.com/paysera/lib-another-resource-client.git",
            "raml_definitions": [
                "app-checkout/another-resource"
            ]
        }
    ]
}
```

* There sould be a script, that could be hooked on git-push, it should be able to:
    * extract MAJOR, MINOR or PATCH strings from commit message
    * decide which client should be updated
    * generate client from configuration
    * bump new version to repository based on extracted semantic version constraint
 
* util-raml-code-generator
    * Utility should be able to accept single client configuration, in case of multiple raml_definitions it should:
        * handle different baseUri inside ClientFactory
        * generate separate *Client classes
        * generate meged README.md
        * handle reference to the same Types/Traits:
        
```        
app-checkout
|--config.json
|--types
|   |--User.raml
|   |--Something.raml
|--some-resource
|   |--api.raml:
|         types:
|           - User: !include ../types/User.raml
|           - Something: !include ../types/Something.raml
|--some-other-resource:
|   |--api.raml:
|         types:
|            - User: !include ../types/User.raml
|            - Something: !include ../types/Something.raml
```

In this case utility should recognize usage of common Types (User, Something) and ensure interoperability between SomeResourceClient and SomeOtherResourceClient
