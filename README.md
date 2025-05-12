# Motivation

Many RO-Crate users are interested in formalising how we define *machine-readable* profiles in a way that allows validation against these profiles.
This is a common subject in the RO-Crate Slack and GitHub, e.g.
https://github.com/ResearchObject/ro-crate/issues/399 and https://github.com/ResearchObject/ro-crate/issues/264.
Additionally, WEHI, ARDC and LDaCA are all working on projects that need formalised schemas.

The use cases for these schemas are typically:

* Validating a given crate against a profile, reporting any failures in conformance
* Building editors and other tools that leverage the schema. For example, Crate-O's mode files.

# Proposal

I propose that we encourage putting a SHACL schema definition within `ro-crate-metadata.json` of the profile crate.

For example, consider the [Process Run Profile](https://www.researchobject.org/workflow-run-crate/profiles/process_run_crate/), which is a great example because it already has validation rules in a human-readable form, and is a relatively small profile.

To support this, we first need to add the SHACL vocabulary to the RO-Crate context:

```json
"@context": {
    "sh": "http://www.w3.org/ns/shacl#",
    "NodeShape": "sh:NodeShape",
    "PropertyShape": "sh:PropertyShape",
    "targetClass": "sh:targetClass",
    "property": "sh:property",
    "path": "sh:path"
}
```

We might choose to "namespace" these terms with a prefix or suffix, in order to clarify that they are specifically for SHACL validation:

```json
"@context": {
    "sh": "http://www.w3.org/ns/shacl#",
    "NodeShapeSh": "sh:NodeShape",
    "PropertyShapeSh": "sh:PropertyShape",
    "targetClassSh": "sh:targetClass",
    "propertySh": "sh:property",
    "pathSh": "sh:path"
}
```

In this profile, a `CreateAction` must have the `instrument` property. We can encode this using SHACL inside the `ro-crate-metadata.json` like this:

```json
[
    {
        "@id": "#CreateActionShape",
        "@type": "NodeShape",
        "targetClass": "CreateAction",
        "property": [
            {"@id": "_:CreateActionInstrumentConstraint"}
        ],
    },
    {
        "@id": "_:CreateActionInstrumentConstraint",
        "@type": "PropertyShape",
        "path": "instrument",
        "minCount": 1
    }
]
```

Unfortunately the SHACL has to be somewhat more verbose [due to the restriction that RO-Crate JSON is flattened](https://www.researchobject.org/ro-crate/specification/1.1/structure.html#ro-crate-metadata-file-ro-crate-metadatajson)

# Why SHACL?

* SHACL is itself is written in RDF, so we don't need a new file containing the schema as we would with say a LinkML schema
* SHACL provides the vocabulary we need for typical validation:
    * Range: [`sh:class`](https://www.w3.org/TR/shacl/#ClassConstraintComponent)
    * Required/optional/cardinality: [`sh:minCount` and `sh:maxCount`](https://www.w3.org/TR/shacl/#MinCountConstraintComponent)
* SHACL is already used in the [`rocrate-validator`](https://github.com/crs4/rocrate-validator)

# Generating other schemas

As mentioned above, one use case for defining machine-readable RO-Crate schemas is to convert them into other forms.
One such form might be a Crate-O mode file.
If we consider the above `CreateAction` example, we can easily find validations for `CreateAction` by looking for the triple `?shape sh:targetClass schema:CreateAction`, for example using SPARQL. 
From there, we can extract constraints and convert them into a mode file.

# Alternatives

## Why not LinkML?

I have previously tested profile definition and generation using LinkML: [`WEHI-SODA-Hub/proclaim`](https://github.com/WEHI-SODA-Hub/proclaim).
My main issues with this approach are [explained here](https://github.com/WEHI-SODA-Hub/proclaim/blob/main/linkml_issues.md). In brief:
* This requires introducing a new file format (YAML), new syntax etc
* RO-Crate concepts such as "data entities" which are not actually classes aren't readily modelled using LinkML's type system
* Profile validations would probably require re-defining LinkML classes. Modelling the validation as standalone constraints are much simpler to reason with.
* LinkML can't easily make statements about the graph itself, such as the existence of certain entities, whereas SHACL can
* LinkML doesn't yet support "extra" properties that are not defined in the schema
* The LinkML SHACL generator is not production ready, meaning that we would have no way to directly *use* the schemas
* Various runtime features are missing from LinkML

## Why not OWL?

OWL would work similarly to SHACL, with many of the same advantages.
However, OWL prefers to apply validations by subclassing existing classes.
This is not desirable because then crates that *use* this profile will have to use those subclasses, which adds a lot of complexity that isn't desired.
For example, the above SHACL constraint could be modelled using OWL like this:

```json
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

Then, users of the profile would need to use `http://example.org/CreateActionWithInstrumentRestriction"` instead of `schema:CreateAction`.
