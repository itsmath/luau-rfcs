# Type Attributes

## Summary

This RFC proposes syntax and semantics for type attributes in Luau. It builds on prior work from the [Attributes (for Functions)](./syntax-attributes-functions.md) RFC and the [Function Attribute Parameters](./syntax-attributes-functions-parameters.md) RFC.

## Motivation

Type attributes allow developers to, for _any specific_ type:

- Change the behavior of the type solver/analyzer regarding that type,
- Change the behavior of external tooling - such as linters and language servers - regarding that type, 
- Add metadata to the type, which can be processed by external tooling or user-defined type functions.

Currently, the biggest use case is going to be the use of the `@deprecated` attribute on types other than functions, but this RFC or another proposal like it needs to be implemented for potential future attributes, such as a `@doc` or `@nodiscard` attribute.

## Design

The RFC proposes four different ways to define a type attribute, with different semantics.

- 1. The default. Should be used if the type is anonymous such as in function returns/parameters.

     ```luau
     local function changeFoo(foo: @attribute Foo) end
     ```

- 2. The type attribute on top of a type.

     ```luau
     @attribute
     type Foo = any -- This is equivalent to @attribute any.
     ```

     Note that while for some attributes this is equivalent to 1., this has different semantics. `@attribute Type` describes an attribute applied on the `Type` itself, while this describes an attribute applied to the type definition, which may carry over to the type, depending on the definition of the attribute. By default, (for example, if the attribute isn't defined by Luau at all), it carries over.

- 3. The type attribute on top of of a field/property/indexer of a table type.

     ```luau
     type Foo = {
          @attribute
          changeFoo: (foo: Foo) -> (), -- This is equivalent to @attribute (foo: Foo) -> ().

          @attribute
          [number]: any, -- This is equivalent to @attribute any.
     }
     ```

     This works in the same way as 2. The semantic meaning of an attribute on top of a field/property/indexer of a table is that the attribute applies to that field/property/indexer, but by default, it carries over to the type.

- 4. The type attribute on top of a function

     ```luau
     @attribute
     function foo() end
     ```

     This is the same as described in the [Attributes (for Functions)](./syntax-attributes-functions.md) RFC. This is the same thing as 2., in that the semantics change, but by default, the attribute is carried over to the type of the function. Note however that depending on the definition of the attribute, it could carry over to other parts of the function too, such as the table (think `function foo.bar`), or the the result type.

Additionally, for the two attributes we have currently, `@deprecated` and `@native`, only `@deprecated` is a _type attribute_, in that it'll actually carry the information from the value to the type, as described in 2., 3., and 4. `@native` is especifically for function definitions, so it must be defined there.

### `type:attributes`

For the following sections, let the following types be defined:

```luau
type Attribute<Name = string, Argument = unknown> = {
     name: Name,
     arguments: {Argument}, -- This should probably be a tuple, but we don't have those in Luau currently
}

-- Note that language-defined type attributes should be unioned here so autocomplete works nicely
type Attributes = {Attribute | Attribute<"deprecated", { reason: string?, use: string? }?>}
```

This method returns all attributes in the given type. This has the same behavior of other functions in the type runtime, such as `type:properties`, where the returned table is simply a copy/view of the data, and changing it has no effect on the actual attributes of the time (unless updated with `type:setattributes`).

The function signature should look something like this:

```luau
function type:attributes(): Attributes end
```

### `type:setattributes`

This method sets the attributes for the given type. You can also pass `nil` to clear all attributes from the type.

The function signature should look something like this:

```luau
function type:setattributes(attributes: Attributes?) end
```

### `type:addattribute`

This method adds an attribute to the given type. This is an utility function.

The function signature should look something like this:

```luau
function type:addattribute(name: string, ...: unknown) end
```

Just like the `Attributes` type defined earlier, it might be beneficial to make this function overloaded:

```luau
type AddAttributeOverload<Name = string, Arguments... = ...unknown> = (name: Name, Arguments...) -> ()

type AddAttribute = AddAttributeOverload
                  & AddAttributeOverload<"deprecated", { reason: string?, use: string? }?>
```

## Drawbacks

Adds additional complexity to the language, which may not be needed (we could continue to support attributes only for functions or values, for example).

## Alternatives

There are a few options:

1. We could not do this at all, and users would continue to have to rely on external tooling (like `luau-lsp` with documentation comments) to convey extra information about a given type. This can end up fragmenting the ecosystem of the language.

2. We could not support attributes in the type runtime, since, for example, it's easy to think that `functiontype:addattribute("native")` would make the function compile natively, when that isn't true at all. This RFC proposes that this confusion between _type_ attributes and _compiler_ attributes will always exist, so we might as well just accept it and document the difference well to prevent silent errors.

3. We could only allow the `@attribute Type` syntax, but that'd make defining a long list of attributes hard to do in a pleasing way, and most 
likely confusing.