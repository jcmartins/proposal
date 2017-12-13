# Proposal: GraphQL Code Generation

Author(s): @jamesdphillips

Last updated: 2017-11-28

Discussion at https://github.com/sensu/sensu-go/issues/490.

## Abstract

Introduce code generation for the GraphQL schema implementation to reduce the
overhead of making additions and help make the schema more friendly to work with
for both internal and external developers.

## Background

Initial implementation of GraphQL endpoint was met with understandable
trepidation with regard to how much code was required to define types. This is
especially true when you consider that a public REST API is a also a hard
requirement; meaning the cost of maintaining two interfaces must remain minimal.

Another concern with the current implementation, that could be tackled
concurrently, is that it is not currently possible to configure a resolver,
type, or schema with the other requisite items. To provide an example, currently
resolvers are dependent on the store being injected in to the context so that it
can fetch data. Ideally groups of resolvers could be configured with the store
(and other sources of data) at start up.

## Proposal

Instead of describing our schema in Go as we were previously, we describe our
schema in GraphQL itself. To facilitate this we write simple tooling that takes
a GraphQL schema definition and generates our Go code.

To give an example, a engineer would introduce a type using GraphQL's schema
language; this is demonstrated below.

graphql/dog.graphql
```graphql
# Dogs make for very good friends.
type Dog {
  name: String!
  breed: DogBreedEnum

  # Is it a puppy? y/n.
  isPuppy: Boolean!

  # The length of the dog in whatever units you please. Defaults to meters.
  length(unit: LengthUnit = METER): Float!

  feet: Int!
  feets: Int @deprecated(reason: "James does not undestand english.")
}

# Breeds a dog could be
enum DogBreedEnum {
  BORDERCOLLIE
  SHIBA
  POM
  POODLE
}
```

Next using proposed tooling, something similar to the following Go code would
be generated.

graphql/dog.gql.go
```golang
package schema

import "githu.com/graphql-go/graphql"

//
// Expose an interface for each type that that developers can implement. In this
// way an implementation of the resolver can be configured with the controllers,
// the store, and other data sources or services the resolver(s) may require.
//

type DogTypeResolver interface {
  Name(graphql.ResolveParams) (interface{}, error)
  Breed(graphql.ResolveParams) (interface{}, error)
  IsPuppy(graphql.ResolveParams) (interface{}, error)
  Length(graphql.ResolveParams) (interface{}, error)
  Feet(graphql.ResolveParams) (interface{}, error)
  Feets(graphql.ResolveParams) (interface{}, error)
}

func NewDogType(resolver DogTypeResolver) *graphql.Object {
  nameField := graphql.Field{
    Name:        "name",
    Type:        graphql.String,
    Description: "Self explainatory",
    Resolve:     resolver.Name,
  }
  breedField := graphql.Field{
    Name:        "breed",
    Type:        BreedEnum, // TBD
    Description: "Self explainatory",
    Resolve:     resolver.Breed,
  }
  isPuppyField := graphql.Field{
    Name:        "isPuppy",
    Type:        graphql.Boolean,
    Description: "Is it a puppy? y/n.",
    Resolve:     resolver.IsPuppy,
  }
  lengthField := graphql.Field{
    Name:        "length",
    Type:        graphql.Float,
    Description: "The length of the dog in whatever units you please. Defaults to meters.",
    Resolve:     resolver.Length,
  }
  feetField := graphql.Field{
    Name:        "feet",
    Type:        graphql.Integer,
    Description: "Self explainatory",
    Resolve:     resolver.Feet,
  }
  feetsField := graphql.Field{
    Name:        "feets",
    Type:        graphql.Integer,
    Description: "Self explainatory",
    Resolve:     resolver.Feets
  }

  return graphql.NewObject(graphql.ObjectConfig{
    Name: "Dog",
    Description: "Dogs make for very good friends.",
    Fields: graphql.FieldsThunk(func() graphql.Fields {
      return graphql.Fields{
        "name":    &nameField,
        "breed":   &breedField,
        "isPuppy": &isPuppField,
        "length":  &lengthField,
        "feets":   &feetsField,
        "feet":    &feetField,
      },
    }),
  })
}
```

After code is generated we simply need to write an implementation of the
Resolver. For what it's worth, this is relatively congruent with how gRPC
operations are implemented.

graphql/dog.go
```golang
type DogResolver struct {
  breedController controllers.BreedController
  status          apid.StatusPb
}

func NewDogResolver(store store.BreedStore) *DogResolver {
  return &DogResolver{
    breedController: controllers.NewBreedController(store),
  }
}

func (r *DogResolver) Breed(p graphql.ResolveParams) (interface{}, error) {
  dog, ok := p.Source.(*Dog)
  if !ok {
    return nil, errors.New("given source was not a dog")
  }

  return := r.breedController.Find(dog.Breed)
}

func (r *DogResolver) Name(p graphql.ResolveParams) (interface{}, error) {
  if dog, ok := p.Source.(*Dog); ok {
    return dog.Name, nil
  }
  return nil, errors.New("given source was not a dog")
}

func (r *DogResolver) IsPuppy(p graphql.ResolveParams) (interface{}, error) {
  if dog, ok := p.Source.(*Dog); ok {
    return dog.AgeInMonths < 12, nil
  }
  return nil, errors.New("given source was not a dog")
}

// So on...
```

Putting it all together...

graphql/service.go
```golang
type ServiceConfig struct {
  store  store.Store
  status apid.StatusPb
  // ...?
}

type Service struct {
  schema   *graphql.Schema
  executor Exectuor
  // ...?
}

func (s *Service) Execute(q string, params map[string]string) *graphql.Results {
  // ...?
  s.executor.Exec(Schema, q, params)
}

func NewService(conf ServiceConfig) Service {
  // ...
  registerType(&schema, NewDogType(NewDogResolver(config.store)))
  registerType(&schema, NewCatType(NewCatResolver(config.store, ...)))
  // ...
}
```

## Rationale

My hope is the above demonstrates a friendlier and more expedient of defining
our GraphQL schema and resolvers. The following is a point form list of some
advantages, disadvantages and alternatives to this approach.

### Advantages

- We no longer have to define the schema in clunky verbose code.
- Can inject dependencies when resolvers are defined instead of inserting them
  into the context. (Eg. our store, external services, etc.)
- This approach is similar to how one might define a gRPC server.
  - Should feel familiar and easy to many existing Go developers.
  - Established pattern that appears to work well for numerous projects.

### Disadvantages

- Requires user to have some understanding of GraphQL's schema definition
  syntax.
- Cannot (easily) share field descriptions between our types package and our
  GraphQL types.
- Can no longer programmatically describe common patterns. For example it may
  become annoying to describe connections continually (refer to Relay's
  [connection specification]).
  - In long run, if we decide it is problematic, we could always write a custom
    directive our tooling implements, that sweeps some of the extraneous bits
    under the rug.

### Alternatives

During initial discussion we had considered using our protocol buffers to
generate our schema. While I think this was a very clever idea and potentially
still workable, after some initial prototyping I think there are a few
fundamental differences that make this approach non-trivial.

- Protocol buffers describe concrete messages between two systems; are less
  concerned with the relationship between one or more entities.
- Non-trivial to describe fields that _should not_ be exposed by front-end;
  would require extension.
- Non-trivial to describe computed fields; would require extension.
  - For instance on my `UserType` it's unlikely that I want to expose the user's
    password digest, however, I may want to expose a field that communicates
    whether or not the user _has_ a password present.
- Types that are specific to front-end implementation require same effort as
  before.
- Custom extensions to make up for deficiencies would be non-standard and would
  require extra documentation. Only Sensu team members would have sufficient
  domain knowledge to use or change.

Despite this we easily could write a simple generator that could take our
existing protobuf definitions and create an initial GraphQL schema definition.Of
course, we would only ever be able to run the operation once for each type, but
at least it would reduce the initial overhead by copying fields and comments
over.

## Compatibility

Rip and tear.

## Implementation

- Refer to [proposal](#proposal) above for demonstration of how developers would
  implement types and resolvers.
- Implementation of tooling would use `graphql-go/graphql/language` package to
  parse documents and iterate over AST to generate Go code.

All implementation subject to change.

## Open issues

Sensu Issue [#490](https://github.com/sensu/sensu-go/issues/490).

## References / Previous Art

- For example of GraphQL schema definition, see generated copy of [Github's schema](https://github.com/gjtorikian/graphql-docs/blob/master/test/graphql-docs/fixtures/gh-schema.graphql).
- For comparison: 'Hello World' [gRPC server](https://github.com/grpc/grpc-go/tree/master/examples/helloworld) implementation.
- Example of previous GraphQL type definition. [Code](https://github.com/sensu/sensu-go/blob/53cb01002ae7bc4dcde841789b90cff3b82694af/backend/apid/graphql/entity.go).

[connection specification]:https://facebook.github.io/relay/docs/graphql-connections.html
