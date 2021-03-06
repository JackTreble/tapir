# Custom types

To support a custom type, you'll need to provide an implicit `Codec` for that type.

This can be done by writing a codec from scratch, mapping over an existing codec, or automatically deriving one.
Which of these approaches can be taken, depends on the context in which the codec will be used.

## Providing an implicit codec

To create a custom codec, you can either directly implement the `Codec` trait, which requires to provide the following
information:

* `encode` and `rawDecode` methods
* optional schema (for documentation)
* optional validator
* codec format (`text/plain`, `application/json` etc.)

This might be quite a lot of work, that's why it's usually easier to map over an existing codec. To do that, you'll 
need to provide two mappings: 

* a `decode` method which decodes the lower-level type into the custom type, optionally reporting decode failures 
(the return type is a `DecodeResult`)
* an `encode` method which encodes the custom type into the lower-level type

For example, to support a custom id type:

```scala
import scala.util._

class MyId private (id: String) {
  override def toString(): String = id
}
object MyId {
  def parse(id: String): Try[MyId] = {
    Success(new MyId(id))
  }
}
```

```scala
import sttp.tapir._
import sttp.tapir.CodecFormat.TextPlain

def decode(s: String): DecodeResult[MyId] = MyId.parse(s) match {
  case Success(v) => DecodeResult.Value(v)
  case Failure(f) => DecodeResult.Error(s, f)
}
def encode(id: MyId): String = id.toString

implicit val myIdCodec: Codec[String, MyId, TextPlain] = 
  Codec.string.mapDecode(decode)(encode)
```

Or, using the type alias for codecs in the TextPlain format and String as the raw value:

```scala
import sttp.tapir.Codec.PlainCodec

implicit val myIdCodec: PlainCodec[MyId] = Codec.string.mapDecode(decode)(encode)
```

```eval_rst
.. note::

  Note that inputs/outputs can also be mapped over. In some cases, it's enough to create an input/output corresponding 
  to one of the existing types, and then map over them. However, if you have a type that's used multiple times, it's 
  usually better to define a codec for that type. 
```

Then, you can use the new codec e.g. to obtain an id from a query parameter or a path segment:

```scala
endpoint.in(query[MyId]("myId"))
// or
endpoint.in(path[MyId])
```

## Automatically deriving codecs

In some cases, codecs can be automatically derived:

* for supported [json](json.md) libraries
* for urlencoded and multipart [forms](forms.md)

Automatic codec derivation usually requires other implicits, such as:

* json encoders/decoders from the json library
* codecs for individual form fields
* schema of the custom type, through the `Schema[T]` implicit

## Schema derivation

For case classes types, `Schema[_]` values are derived automatically using [Magnolia](https://propensive.com/opensource/magnolia/), given
that schemas are defined for all of the case class's fields. It is possible to configure the automatic derivation to use
snake-case, kebab-case or a custom field naming policy, by providing an implicit `sttp.tapir.generic.Configuration` value:

```scala
import sttp.tapir.generic.Configuration

implicit val customConfiguration: Configuration =
  Configuration.default.withSnakeCaseMemberNames
```

Alternatively, `Schema[_]` values can be defined by hand, either for whole case classes, or only for some of its fields.
For example, here we state that the schema for `MyCustomType` is a `String`:

```scala
import sttp.tapir._

case class MyCustomType()
implicit val schemaForMyCustomType: Schema[MyCustomType] = Schema(SchemaType.SString)
```

If you have a case class which contains some non-standard types (other than strings, number, other case classes, 
collections), you only need to provide the schema for the non-standard types. Using these schemas, the rest will
be derived automatically.

### Sealed traits / coproducts

Tapir supports schema generation for coproduct types (sealed trait hierarchies) of the box, but they need to be defined
by hand (as implicit values). To properly reflect the schema in [OpenAPI](../openapi.md) documentation, a
discriminator object can be specified. 

For example, given following coproduct:

```scala
sealed trait Entity {
  def kind: String
} 
case class Person(firstName:String, lastName:String) extends Entity { 
  def kind: String = "person"
}
case class Organization(name: String) extends Entity {
  def kind: String = "org"  
}
```

The schema may look like this:

```scala
import sttp.tapir._

val sPerson = implicitly[Schema[Person]]
val sOrganization = implicitly[Schema[Organization]]
implicit val sEntity: Schema[Entity] = 
    Schema.oneOf[Entity, String](_.kind, _.toString)("person" -> sPerson, "org" -> sOrganization)
```

## Customising derived schemas

In some cases, it might be desirable to customise the derived schemas, e.g. to add a description to a particular
field of a case class. This can be done by looking up an implicit instance of the `Derived[Schema[T]]` type, 
and assigning it to an implicit schema. When such an implicit `Schema[T]` is in scope will have higher priority 
than the built-in low-priority conversion from `Derived[Schema[T]]` to `Schema[T]`.

Schemas for products/coproducts (case classes and case class families) can be traversed and modified using
`.modify` method. To traverse collections, use `.each`.

For example:

```scala
import sttp.tapir._
import sttp.tapir.generic.Derived

case class Basket(fruits: List[FruitAmount])
case class FruitAmount(fruit: String, amount: Int)
implicit val customBasketSchema: Schema[Basket] = implicitly[Derived[Schema[Basket]]].value
      .modify(_.fruits.each.amount)(_.description("How many fruits?"))
```

There is also an unsafe variant of this method, but it should be avoided in most cases. 
The "unsafe" prefix comes from the fact that the method takes a list of strings, 
which represent fields, and the correctness of this specification is not checked.

Non-standard collections can be unwrapped in the modification path by providing an implicit value of `ModifyFunctor`.

## Next

Read on about [validation](validation.md).
