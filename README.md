# Introduction

When the reflection capabilities of a language is in discussion, the ability to obtain and manipulate names is probably one of the first things that comes to mind.
C++ in it's current form does not provide such functionality. Some approximation can be achieved by using the pre-processor at the cost of giving up on the type-safety.
Other characteristics of reflection (or introspection) are present in the C++ language in the form of type traits which are a good starting point.

Proposals to bring reflection to C++ exist and what they aim to offer in terms of functionality put C++ on par with other languages.
However, all the proposals presented so far:
- Seem to miss the importance of the names in programming. Manipulating names is not the main goal.
- Depart at different levels from ideas of the existing type traits. They offer their own versions.
- Do not offer a way to replace the pre-processor-based solutions for reflection.
- Initially targeted torwards the more advanced user of the language. Even for the simple use-cases.

These are the main motivations behind this proposal.

# Reasoning About Names

An excellent material about how important are names in programming and how the C++ language should handle names can be found at:
https://github.com/rbock/reasoning-about-names/blob/master/README.md

It introduces the concept of the type ```std::name```, that represents the name of an entiy in the code.
This proposal aims to be the other half of that proposal.

## Static Reflection

Similarly to the other proposals presented so far, this one introduces a form of static reflection.

# Language Additions

This proposal introduces the following constructions to the language:
- A literal ```std::name``` operator: tentatively `` (backticks)
- A ```std::name```-to-entity operator: tentatively, ```@``` (the at sign)
- Name traits, along with a new header ```<name_traits>```

## Literal ```std::name``` operator

This operator takes a name as defined in section [3 - Basic] of the standard and returns a constexpr instance of ```std::name```.
The contents of name refer to the fully qualified name of the referred entity, for example:
```C++
namespace N {
  struct S {
    static int i;
  };
  constexpr auto name_of_S = `S`; // std::name = `::N::C`
  constexpr auto name_of_i = `S::i`; // std::name = `::N::C::i`
}
```

The provided the name can be qualified or not.
```::``` is accepted as the way to obtain the name of the global scope.

## ```std::name```-to-entity operator

This operator takes a ```std::name``` instance and replace it by the referred entity.
Based on the previous example, the following should be possible:
```C++
template <std::name StaticMemberName>
struct StaticGetter {
  decltype(auto) get() {
    return @StaticMemberName;
  }
};
```

## The ```<name_traits>``` header

In this header lies most of the functionality needed for support static reflection.
The traits can be split up into three main sections:
- ```std::name``` facilities.
- ```std::name``` categories.
- ```std::name``` instrospection.

It's important to mention that most of the traits below can only be implementent with the support of compiler built-ins.

### Facilities

In this section of the ```<name_traits>``` header we find the following traits:
- ```constexpr std::name std::to_unqualified_name_v<std::name>```
- ```constexpr std::name std::to_qualified_name_v<std::name>```
- ```constexpr std::name[] std::get_name_qualifiers_v<std::name>```
- ```constexpr std::name std::compose_name<std::name...>```
- ```constexpr std::source_location std::name_declaration_v<std::name>```

#### ```constexpr std::name std::to_unqualified_name_v<std::name>```

Given a name, returns its unqualified version.
For example:
```C++
constexpr auto unq_name_of_i = std::to_unqualified_name_v<name_of_i>; // Simply `i`
```

If the name is already unqualified, returns the input.

#### ```constexpr std::name std::to_unqualified_name_v<std::name>```

Given a name, returns its qualified version.
For example:
```C++
constexpr auto q_name_of_i = std::to_unqualified_name_v<unq_name_of_i>; // `::N::S::i`
```

Please note that qualifying a name is subject to the name lookup rules in effect at the point of the call.

#### ```constexpr std::name[] std::get_name_qualifiers_v<std::name>```

Given a name, returns the unqualifed names of the parent scopes in an array.
The order is from the immediate parent scope to the global scope.
For example:
```C++
constexpr auto parents_of_i = std::get_name_qualifiers_v<name_of_i>; // [`S`, `N`, `::`]
```

If the passed name is unqualified, it will be made qualified first.

#### ```constexpr std::name std::compose_name_v<std::name...>```

Given an array of names, try to compose the qualified name of an existing entity.
More formally:
```C++
template <std::name... Names>
using compose_name_v = std::compose_name<Names...>::value;
```
If the composed name does not refer to any entity, the compilation fails with diagnostics.
Example:
```C++
constexpr auto made_up_member_of_j_error = std::compose_name_v<
  $"j", // Literal std::name
  parents_of_i[0],
  parents_of_i[1],
  parents_of_i[2]>; // Error: 'j' is not a member of 'S'
```

If a passed name in the array is qualified, it will be made unqualified first.

#### ```constexpr std::source_location std::name_declaration_v<std::name>```

Given a name, returns the source_location where the name is declarated/defined (what comes last).
For example:
```C++
constexpr auto location = std::name_declaration_v<name_of_i>;
```

### Categories

In this section of the ```<name_traits>``` header we find the following traits:
- ```constexpr bool std::is_qualified_name_v<std::name>``` - ```true``` if the given name is fully qualified.
- ```constexpr bool std::is_unqualified_name_v<std::name>``` - ```true``` if the given name is unqualified.
- ```constexpr bool std::is_scope_name_v<std::name>``` - ```true``` if it's the name of a scope (such as the name of a namespace, class, struct, union, enum, global).
- ```constexpr bool std::is_internal_name_v<std::name>``` - ```true``` if it's an internal (implementation-defined name, such as unammed structs, unions of lambdas).
- ```constexpr bool std::is_object_name_v<std::name>``` - ```true``` if it's the name of an object (member or variable or reference).
- ```constexpr bool std::is_type_name_v<std::name>``` - ```true``` if it's the name of a type, typedef or ```using``` alias (in the examples above, ```name_of_S```).
- ```constexpr bool std::is_function_name_v<std::name>``` - ```true``` if it's the name of a single function or an overload set.
- ```constexpr bool std::is_operator_name_v<std::name>``` - ```true``` if it's the name of a single operator or an overload set (i.e. `operator++`).
- ```constexpr bool std::is_conversion_name_v<std::name>``` - ```true``` if it's the name of a conversion operator (i.e. `operator int`).
- ```constexpr bool std::is_argument_name_v<std::name>``` - ```true``` if it's the name of a function argument.
- ```constexpr bool std::is_namespace_name_v<std::name>``` - ```true``` if it's the name of a namespace.
- ```constexpr bool std::is_enumerator_name_v<std::name>``` - ```true``` if it's the name of a enumeration value.
- ```constexpr bool std::is_bitfield_name_v<std::name>``` - ```true``` if it's the name of a class bitfield.

Other possibilities:
- ```constexpr bool std::is_reserved_name_v<std::name>``` - ```true``` if it's a keyword, a reserved name as described in section [2.10 - Identifiers] of the standard.
- ```constexpr bool std::is_reserved_keyword_v<std::name>``` - ```true``` if it's a keyword or identifiers with special meaning (final, override).
- ```constexpr bool std::is_storage_specifier_v<std::name>``` - ```true``` if it's one of [7.1.1 - Storage Class Specifiers] defined in the standard.
- ```constexpr bool std::is_function_specifier_v<std::name>``` - ```true``` if it's one of [7.1.2 - Function Specifiers] defined in the standard.
- ```constexpr bool std::is_typedef_specifier_v<std::name>``` - ```true``` if it's [7.1.3 - Typedef Specifier] defined in the standard.
- ```constexpr bool std::is_friend_specifier_v<std::name>``` - ```true``` if it's [7.1.4 - Friend Specifier] defined in the standard.
- ```constexpr bool std::is_constexpr_specifier_v<std::name>``` - ```true``` if it's [7.1.5 - Constexpr Specifier] defined in the standard.
- ```constexpr bool std::is_cv_qualifier_v<std::name>``` - ```true``` if it's one of [7.1.6.1 The cv-qualifiers] defined in the standard.
- ```constexpr bool std::is_access_specifier_v<std::name>``` - ```true``` if it's one of [11.1 Access specifiers] defined in the standard.
- ```constexpr bool std::is_linkage_specifier_v<std::name>``` - ```true``` if it's `extern`, `extern "C"`, `static`.
- ```constexpr bool std::is_alias_specifier_v<std::name>``` - ```true``` if it's `alias`.
- ```constexpr bool std::is_auto_specifier_v<std::name>``` - ```true``` if it's `auto`.
- ```constexpr bool std::is_label_name_v<std::name>``` - ```true``` if it's the of a goto or case label.
- ```constexpr bool std::is_entity_name_v<std::name>``` -  ```false``` if it's the of a goto or case label.
- ```constexpr bool std::is_special_meaning_v<std::name>``` - ```true``` if it's `final`, `override` and unqualified.

In the future:
- ```constexpr bool std::is_template_name_v<std::name>```
- ```constexpr bool std::is_template_argument_name_v<std::name>```
- ```constexpr bool std::is_concept_name_v<std::name>```

### Instrospection

In this section of the ```<name_traits>``` header we find the following traits:
- ```constexpr std::name[] std::get_declared_names_v<std::name>```
- ```constexpr std::name[] std::get_object_specifiers_v<std::name>```
- ```constexpr std::name[] std::get_type_specifiers_v<std::name>```
- ```typelist<Signatures...> std::get_overloads_t<std::name>```
- ```constexpr std::name[] std::get_function_specifiers_v<Signature, std::name>```
- ```constexpr std::name[] std::get_arguments_v<Signature, std::name>```

For the examples below, have the following code as references.
```C++
class C {
public:
  typedef int Int;
public:
  C(int value = 0) : i(value == 0 ? j : value) {}
  virtual Int get_value() const { return i; }
  void set_value(Int value) { i = value; }
  void set_value(bool value) { i = value ? 1 : 0; }
protected:
  static void set_default(Int value = 0) { C::j = value; }
private:
  static Int j;
protected:
  mutable Int i;
};
```

#### ```constexpr std::name[] std::get_declared_names_v<std::name>```

Given a scope name and an optional list of name traits, returns an array containing all the declarared names meeting the criteria.
More formally:
```C++
template <std::name Name, template <std::name> class ...Filter>
using get_declared_names_v = std::get_declared_names<std::name, Filter...>::value;
```
Examples:
```C++
constexpr auto data_members_of_C = std::get_declared_names_v<`C`, std::is_object_name_v>; // `j`, `i`
constexpr auto member_funcs_of_C = std::get_declared_names_v<`C`, std::is_function_name_v>; // `C`, `~C`, `get_value`, `set_value`, `set_default`
constexpr auto nested_types_of_C = std::get_declared_names_v<`C`, std::is_type_name_v>; // `Int`
template <std::name Name>
struct IsJ {
  constexpr bool value = std::to_unqualified_name_v<Name> == $"j";
};
template <std::name Name>
using IsJ_v = IsJ<Name>::value;
constexpr auto j_of_C = std::get_declared_names_v<`C`, std::is_object_name_v, IsJ_v>; // `j`
```

Please note that the compilation fails if the name does refer to a scope.

#### ```constexpr std::name[] std::get_object_specifiers_v<std::name>```

Given an object name and an optional list of name traits, returns an array containing all the visible specifiers associated with the name.
More formally:
```C++
template <std::name Name, template <std::name> class ...Filter>
using get_object_specifiers_v = std::get_object_specifiers<std::name, Filter...>::value;
```
Examples:
```C++
constexpr auto j_specifiers = std::get_object_specifiers_v<`C::j`>; // `private`, `static`
constexpr auto i_specifiers = std::get_object_specifiers_v<`C::i`>; // `protected`, `mutable`
constexpr auto i_access = std::get_object_specifiers_v<`C::i`, std::is_access_specifier_v>; // `protected`
constexpr auto i_is_public = i_access[0] == $"public"; // false
```

Please note that the compilation fails if the name does refer to an object.

#### ```constexpr std::name[] std::get_type_specifiers_v<std::name>```

Given a type name and an optional list of name traits, returns an array containing all the visible specifiers associated with the name.
More formally:
```C++
template <std::name Name, template <std::name> class ...Filter>
using get_type_specifiers_v = std::get_type_specifiers<std::name, Filter...>::value;
```
Examples:
```C++
constexpr auto Int_specifiers = std::get_type_specifiers_v<`C::Int`>; // `private`, `typedef`
constexpr auto Int_typedef = std::get_object_specifiers_v<`C::i`, std::is_typedef_specifier_v>; // `typedef`
constexpr auto Int_is_typedef = Int_typedef[0] == $"typedef"; // true
```

Please note that the compilation fails if the name does refer to a type.

#### ```typelist<Signatures...> std::get_overloads_t<std::name>```

Given the name of an overload set (or function name), obtains the types of the function signatures.
More formally:
```C++
template <std::name OverloadSet>
struct get_overloads {
  template <typename ...Signatures> // compiler-generated
  using type = typelist<Signatures...>;
};
template <std::name OverloadSet>
using get_overloads_t = std::get_overloads<OverloadSet>::type;
```
Examples:
```C++
using set_value_overloads = std::get_overloads_t<`C::set_value`>; // typelist<void(int), void(bool)>
using set_value_first_overload = nth_element_t<0, set_value_overloads>; // an equivalent of std::tuple_element
auto pointer_to_first_set_value = static_cast<set_value_first_overload*>(&C::set_value);

using C_constructor_overloads = std::get_overloads_t<`C::C`>; // typelist<void(int)>
```

#### ```constexpr std::name[] std::get_function_specifiers_v<Signature, std::name>```

Given an overload set name, a function signature type and an optional list of name traits, returns an array containing all the visible specifiers associated with the name.
More formally:
```C++
template <typename Signature, std::name Name, template <std::name> class ...Filter>
using get_function_specifiers_v = std::get_function_specifiers<Signature, std::name, Filter...>::value;
```
Examples:
```C++
constexpr auto set_default_specifiers = std::get_function_specifiers_v<void(int), `C::set_default`>; // `protected`, `static`
constexpr auto get_value_specifiers = std::get_function_specifiers_v<int(), `C::get_value`, std::is_function_specifier_v>; // `virtual`
constexpr auto get_value_is_virtual = Int_typedef[0] == $"virtual"; // true
```

#### ```constexpr std::name[] std::get_arguments_v<Signature, std::name>```

Given an overload set name, a function signature type obtain the names of the argments.
More formally:
```C++
template <typename Signature, std::name Name>
using get_arguments_v = std::get_arguments<Signature, std::name>::value;
```
Example:
```C++
constexpr auto C_ctor_args = std::get_arguments_v<void(int), `C::C`>; // `value`
```

## Missing Pieces

- ```default``` operator - tentative - obtain the value default initializer of a const, static, function arguments, or type default constructor.
- Template instrospection - pending, dependent of some non-existing language feature, such as deduced non-type parameters - A.K.A. ```template <auto value> struct X{}```.
- Mechanism for access violation - pending, eiter a cast operator ```unrestricted_access(C::j)``` or inheritance-based ```class X : friend Y {};```
- Inheritance type traits - pending, such as ```typelist<Bases...> std::bases_of_t<T>```
- More examples & How to guides.
