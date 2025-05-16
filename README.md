# Proposal


# Motivation

Many RO-Crate users are interested in formalising how we define
*machine-readable* profiles in a way that allows validation against
these profiles. This is a common subject in the RO-Crate Slack and
GitHub, e.g. <https://github.com/ResearchObject/ro-crate/issues/399> and
<https://github.com/ResearchObject/ro-crate/issues/264>. Additionally,
WEHI, ARDC and LDaCA are all working on projects that need formalised
schemas.

The use cases for these schemas are typically:

- Validating a given crate against a profile, reporting any failures in
  conformance
- Building editors and other tools that leverage the schema. For
  example, Crate-O’s mode files.

In short, I propose that we encourage putting a SHACL schema definition
within `ro-crate-metadata.json` of the profile crate.

For example, consider the [Process Run
Profile](https://www.researchobject.org/workflow-run-crate/profiles/process_run_crate/),
which is a great example because it already has validation rules in a
human-readable form, and is a relatively small profile. In this profile,
a `CreateAction` must have the `instrument` property.

## Context

To support this, we first need to add the SHACL vocabulary to the
RO-Crate context:

``` json
{
    "@context": [
        "https://w3id.org/ro/crate/1.1/context",
        {
            "sh": "http://www.w3.org/ns/shacl#",
            "NodeShape": "sh:NodeShape",
            "PropertyShape": "sh:PropertyShape",
            "message": "sh:message",
            "minCount": "sh:minCount",
            "property": "sh:property",
            "parameter": "sh:parameter",
            "Parameter": "sh:Parameter",
            "path": "sh:path",
            "targetNode": "sh:targetNode",
            "targetClass": "sh:targetClass",
            "inversePath": "sh:inversePath"
        }
    ]
}
```

We might choose to “namespace” these terms with a prefix, suffix or
alias, in order to clarify that they are specifically for SHACL
validation. For example, there is a clash between SHACL’s `path`
property and the existing `path` alias that RO-Crate points to
`http://schema.org/contentUrl`.

## Defining the Schema

Next, we can encode this using SHACL inside the `ro-crate-metadata.json`
like this:

``` json
{
    "@context": "../context.jsonld",
    "@graph": [
        {
            "@id": "#CreateActionShape",
            "@type": "NodeShape",
            "targetClass": { "@id": "schema:CreateAction" },
            "property": { "@id": "_:CreateActionInstrumentConstraint" }
        },
        {
            "@id": "_:CreateActionInstrumentConstraint",
            "@type": "PropertyShape",
            "path": { "@id": "schema:instrument" },
            "minCount": 1
        }
    ]
}
```

Following the Process Run Profile, `#CreateActionShape` defines a
“shape” for validating all `CreateAction` instances. This shape links to
`_:CreateActionInstrumentConstraint` via `property`. Then, this
`PropertyShape` applies to the `instrument` property via `path`. It
requires at least one such property exists, using `minCount`.

Unfortunately this SHACL has to be somewhat more verbose than in TTL
format [due to the restriction that RO-Crate JSON is
flattened](https://www.researchobject.org/ro-crate/specification/1.1/structure.html#ro-crate-metadata-file-ro-crate-metadatajson)

## Example Validation

Next, we need an example document that should fail the validation:

``` json
{
    "@context": "../context.jsonld",
    "@graph": [
        {
            "@id": "#invalid_create_action",
            "@type": "CreateAction"
        }
    ]
}
```

Applying the validation using
[`pySHACL`](https://github.com/RDFLib/pySHACL) will then report:

    Validation Report
    Conforms: False
    Results (1):
    Constraint Violation in MinCountConstraintComponent (http://www.w3.org/ns/shacl#MinCountConstraintComponent):
        Severity: sh:Violation
        Source Shape: [ rdf:type sh:PropertyShape ; sh:minCount Literal("1", datatype=xsd:integer) ; sh:path schema1:instrument ]
        Focus Node: <file:///Users/milton.m/Programming/RoCrateProfiles/instrument/invalid.jsonld#invalid_create_action>
        Result Path: schema1:instrument
        Message: Less than 1 values on <file:///Users/milton.m/Programming/RoCrateProfiles/instrument/invalid.jsonld#invalid_create_action>->schema1:instrument

In contrast, we know that a valid crate would look more like this:

``` json
{
    "@context": "../context.jsonld",
    "@graph": [
        {
            "@id": "#invalid_create_action",
            "@type": "CreateAction",
            "instrument": "#some_instrument"
        }
    ]
}
```

Which no longer violates the SHACL constraints:

    Validation Report
    Conforms: True

# Why SHACL?

- SHACL is itself is written in RDF, so it can live inside
  `ro-crate-metadata.json`, and we don’t need a new file containing the
  schema as we would with say a LinkML schema
- SHACL provides the vocabulary we need for typical validation:
  - Range:
    [`sh:class`](https://www.w3.org/TR/shacl/#ClassConstraintComponent)
  - Required/optional/cardinality: [`sh:minCount` and
    `sh:maxCount`](https://www.w3.org/TR/shacl/#MinCountConstraintComponent)
  - Many others [listed
    here](https://www.w3.org/TR/shacl/#core-components)
- SHACL is already used in the
  [`rocrate-validator`](https://github.com/crs4/rocrate-validator)

# Generating other schemas

As mentioned above, one use case for defining machine-readable RO-Crate
schemas is to convert them into other forms. One such form might be a
Crate-O mode file. If we consider the above `CreateAction` example, we
can easily find validations for `CreateAction` by looking for the triple
`?shape sh:targetClass schema:CreateAction`, for example using SPARQL.
From there, we can extract constraints and convert them into a mode
file.

# More Examples

## Combining RDFS with SHACL

If we want to follow [the advice to use `rdfs`
schemas](https://www.researchobject.org/ro-crate/specification/1.2-DRAFT/profiles.html#extension-terms)
in the RO-Crate profiles specification, we can concisely combine these
with SHACL using the [“Implicit Class
Targets”](https://www.w3.org/TR/shacl/#implicit-targetClass) feature of
SHACL. For example, the above `CreateAction` example could be
represented as:

``` json
{
    "@context": "../context.jsonld",
    "@graph": [
        {
            "@id": "schema:CreateAction",
            "@type": [ "rdfs:Class", "NodeShape" ],
            "subClassOf": { "@id": "Action" },
            "property": { "@id": "_:CreateActionInstrumentConstraint" }
        },
        {
            "@id": "_:CreateActionInstrumentConstraint",
            "@type": "PropertyShape",
            "path": { "@id": "schema:instrument" },
            "minCount": 1
        }
    ]
}
```

When applied to the same data in the first example, this correctly
identifies that `instrument` is missing:

    Validation Report
    Conforms: False
    Results (1):
    Constraint Violation in MinCountConstraintComponent (http://www.w3.org/ns/shacl#MinCountConstraintComponent):
        Severity: sh:Violation
        Source Shape: [ rdf:type sh:PropertyShape ; sh:minCount Literal("1", datatype=xsd:integer) ; sh:path schema1:instrument ]
        Focus Node: <file:///Users/milton.m/Programming/RoCrateProfiles/instrument/invalid.jsonld#invalid_create_action>
        Result Path: schema1:instrument
        Message: Less than 1 values on <file:///Users/milton.m/Programming/RoCrateProfiles/instrument/invalid.jsonld#invalid_create_action>->schema1:instrument

## Counting Entities

A common constraint that may be difficult to represent without using
SHACL is that we have one or more entities with a given type. For
example, in a bioimaging crate, we expect to see at least one
`ImageObject` and there has likely been a mistake if no such entity
exists. This is not trivial using a standard validation mechanism,
because it’s not a constraint about each instance of a given class.
[Following this
example](https://www.w3.org/wiki/SHACL/Examples#Counting_Instances),
such a validation can be represented like this:

``` json
{
  "@context": "../context.jsonld",
  "@graph": [
    {
      "@id": "#CountImagesNode",
      "@type": "NodeShape",
      "targetNode": { "@id": "schema:ImageObject" },
      "property": { "@id": "_:CountImageProp" }
    },
    {
      "@id": "_:CountImageProp",
      "@type": "PropertyShape",
      "path": { "@id": "_:CountImagePath" },
      "minCount": 1,
      "message": "The graph must have at least one ImageObject"
    },
    {
      "@id": "_:CountImagePath",
      "@type": "Thing",
      "inversePath": { "@id": "rdf:type" }
    }
  ]
}
```

When applied to a crate without any images:

    Validation Report
    Conforms: False
    Results (1):
    Constraint Violation in MinCountConstraintComponent (http://www.w3.org/ns/shacl#MinCountConstraintComponent):
        Severity: sh:Violation
        Source Shape: [ rdf:type sh:PropertyShape ; sh:message Literal("The graph must have at least one ImageObject") ; sh:minCount Literal("1", datatype=xsd:integer) ; sh:path [ rdf:type schema1:Thing ; sh:inversePath rdf:type ] ]
        Focus Node: schema1:ImageObject
        Result Path: [ rdf:type schema1:Thing ; sh:inversePath rdf:type ]
        Message: The graph must have at least one ImageObject

But applied to a crate containing images:

    Validation Report
    Conforms: True

# Alternatives

## Why not LinkML?

I have previously tested profile definition and generation using LinkML:
[`WEHI-SODA-Hub/proclaim`](https://github.com/WEHI-SODA-Hub/proclaim).
My main issues with this approach are [explained
here](https://github.com/WEHI-SODA-Hub/proclaim/blob/main/linkml_issues.md).
In brief:

- This requires introducing a new file format (YAML), new syntax etc
- RO-Crate concepts such as “data entities” which are not actually
  classes aren’t readily modelled using LinkML’s type system
- Profile validations would probably require re-defining LinkML classes.
  Modelling the validation as standalone constraints are much simpler to
  reason with.
- LinkML can’t easily make statements about the graph itself, such as
  the existence of certain entities, whereas SHACL can
- LinkML doesn’t yet support “extra” properties that are not defined in
  the schema
- The LinkML SHACL generator is not production ready, meaning that we
  would have no way to directly *use* the schemas
- Various runtime features are missing from LinkML

## Why not OWL?

OWL would work similarly to SHACL, with many of the same advantages.
However, OWL prefers to apply validations by subclassing existing
classes. This is not desirable because then crates that *use* this
profile will have to use those subclasses, which adds a lot of
complexity that isn’t desired. For example, the above SHACL constraint
could be modelled using OWL like this:

``` json
{
  "@context": {
    "rdf": "http://www.w3.org/1999/02/22-rdf-syntax-ns#",
    "rdfs": "http://www.w3.org/2000/01/rdf-schema#",
    "owl": "http://www.w3.org/2002/07/owl#",
    "schema": "http://schema.org/",
    "xsd": "http://www.w3.org/2001/XMLSchema#"
  },
  "@id": "http://example.org/CreateActionWithInstrumentRestriction",
  "@type": "owl:Class",
  "rdfs:subClassOf": [
    {
      "@type": "owl:Restriction",
      "owl:onProperty": {
        "@id": "schema:instrument"
      },
      "owl:minQualifiedCardinality": 1,
      "owl:onClass": {
        "@id": "schema:Thing"
      }
    },
    {
      "@id": "schema:CreateAction"
    }
  ]
}
```

Then, users of the profile would need to use
`http://example.org/CreateActionWithInstrumentRestriction` instead of
`schema:CreateAction`.
