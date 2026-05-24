# Symfony Entities - Complete Guide

## Table of Contents

1. [What Is an Entity?](#what-is-an-entity)
2. [Entity as a Plain PHP Class](#entity-as-a-plain-php-class)
3. [Properties and Typed Properties](#properties-and-typed-properties)
4. [Constructors in Entities](#constructors-in-entities)
5. [Getters and Setters](#getters-and-setters)
6. [Fluent Interface (Method Chaining)](#fluent-interface)
7. [PHP 8 Features Used in Entities](#php8-features)
8. [Nullable Properties and Optional Fields](#nullable-properties)
9. [Default Values](#default-values)
10. [Read-Only Properties](#readonly-properties)
11. [Value Objects](#value-objects)
12. [Embeddables](#embeddables)
13. [Entity Relationships (Structure Overview)](#relationships)
14. [Collections in Entities](#collections)
15. [Entity Lifecycle and State](#lifecycle)
16. [DTOs vs Entities](#dtos-vs-entities)
17. [Entity Design Patterns and Best Practices](#best-practices)
18. [Common Mistakes](#common-mistakes)

---

## 1. What Is an Entity? {#what-is-an-entity}

In Symfony and in general software design, an **entity** is a class that represents a real-world concept in your application that has a distinct identity. This identity is what makes it an entity rather than a simple value.

For example, a `User` is an entity because every user is uniquely identifiable regardless of whether their name or email changes. Two users named "Alice" are still two different users. Contrast this with a `Color` or a `Money` value — two `Money` objects representing `100 EUR` are interchangeable, because they are defined only by their value, not by who they are.

An entity in Symfony is nothing more than a regular PHP class. There is no requirement to extend a base class or implement any interface. The "entity" concept is a design decision about how you model your domain. Doctrine (Symfony's default ORM) can later map that class to a database table, but the class itself does not know or care about that.

The key characteristics of an entity are:

- It has a unique identity (usually an `id` property)
- Its identity persists over time even if its properties change
- It encapsulates behavior related to that concept, not just data

---

## 2. Entity as a Plain PHP Class {#entity-as-a-plain-php-class}

Here is the simplest possible entity: a plain PHP class with no dependencies, no parent class, and no interface:

```php
namespace App\Entity;

class Product
{
    private ?int $id = null;
    private string $name;
    private float $price;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function setName(string $name): void
    {
        $this->name = $name;
    }

    public function getPrice(): float
    {
        return $this->price;
    }

    public function setPrice(float $price): void
    {
        $this->price = $price;
    }
}
```

This is a completely valid entity. It has no Doctrine attribute, no validation constraint, no framework dependency whatsoever. You can instantiate it anywhere, use it in tests, pass it between services, and serialize it — all without any framework knowledge.

When you eventually add Doctrine to this class, you simply annotate it with `#[ORM\Entity]` and `#[ORM\Column]` attributes. The class itself does not change structurally.

---

## 3. Properties and Typed Properties {#properties-and-typed-properties}

### Typed vs Untyped Properties

PHP 7.4 introduced typed properties, and Symfony entities make heavy use of them. Typed properties are declared with a type hint directly on the property, not just on the getter/setter.

```php
class User
{
    // Typed properties - PHP knows the type at the property level
    private int $id;
    private string $username;
    private string $email;
    private bool $isActive;
    private float $balance;
    private \DateTimeImmutable $createdAt;
}
```

Without typed properties (older style), every property defaults to `null` until assigned. With typed properties, accessing an uninitialized typed property throws an `Error`. This is actually a good thing — it forces you to think about initialization.

### Common Property Types in Entities

```php
class Article
{
    private ?int $id = null;                        // nullable integer (ID before persistence)
    private string $title;                          // required string
    private ?string $subtitle = null;               // optional string
    private string $content = '';                   // string with default
    private bool $published = false;                // boolean with default
    private int $viewCount = 0;                     // integer with default
    private float $rating = 0.0;                    // float with default
    private \DateTimeImmutable $createdAt;          // immutable datetime
    private ?\DateTimeImmutable $publishedAt = null; // nullable datetime
    private array $tags = [];                       // array with default
}
```

### Why `?int` for the ID

The `id` is typically `?int` (nullable integer) because:

- Before the entity is persisted to the database, it has no ID. The database generates the ID on insert.
- After persistence, the ID is populated by Doctrine.
- Using `?int` makes it clear that the ID may or may not exist.

```php
$product = new Product();
var_dump($product->getId()); // null — not yet persisted

// After saving to the database, getId() returns an integer
```

---

## 4. Constructors in Entities {#constructors-in-entities}

### The Default Constructor

Many entities have no constructor at all, relying on setters to populate properties. This is fine but leaves the object in an incomplete state after instantiation.

### Required Properties via Constructor

A better approach is to use the constructor to enforce required properties, making it impossible to create an invalid entity:

```php
class User
{
    private ?int $id = null;
    private string $email;
    private string $username;
    private \DateTimeImmutable $createdAt;
    private bool $active = true;

    public function __construct(string $email, string $username)
    {
        $this->email     = $email;
        $this->username  = $username;
        $this->createdAt = new \DateTimeImmutable();
    }
}

// Usage — you cannot create a User without email and username
$user = new User('alice@example.com', 'alice');
```

This is called **constructor injection** at the domain level. The entity enforces its own invariants: you cannot have a `User` without an email.

### Named Constructors (Factory Methods)

Sometimes you want multiple ways to create an entity. Named constructors are static factory methods that provide different creation paths:

```php
class Invoice
{
    private ?int $id = null;
    private string $number;
    private string $status;
    private \DateTimeImmutable $issuedAt;
    private ?\DateTimeImmutable $paidAt = null;

    private function __construct(string $number, string $status)
    {
        $this->number   = $number;
        $this->status   = $status;
        $this->issuedAt = new \DateTimeImmutable();
    }

    public static function create(string $number): self
    {
        return new self($number, 'draft');
    }

    public static function createPaid(string $number): self
    {
        $invoice = new self($number, 'paid');
        $invoice->paidAt = new \DateTimeImmutable();
        return $invoice;
    }
}

$draft  = Invoice::create('INV-001');
$paid   = Invoice::createPaid('INV-002');
```

The private constructor prevents arbitrary instantiation. Only the named constructors can create an `Invoice`, and each one guarantees a valid initial state.

### Constructors and Doctrine

Doctrine instantiates entities using reflection and does not call the constructor by default when loading from the database. This means your constructor logic (like setting `createdAt`) does not run on hydration. This is correct behavior — Doctrine restores the entity's state from the database, it does not "create" a new one.

However, if you have initialization logic that must run even on hydration (like initializing a collection), you use the `#[ORM\PostLoad]` lifecycle callback or initialize it in the property declaration instead:

```php
private Collection $comments;

public function __construct()
{
    $this->comments = new ArrayCollection();
}
```

Doctrine calls `__construct()` only when you call `new Entity()` in your code, not when loading from the database. For collections, Doctrine replaces the property with its own proxy collection after loading anyway, so the `ArrayCollection` initialized in the constructor is just a safe default for when the entity is used outside of Doctrine.

---

## 5. Getters and Setters {#getters-and-setters}

### Basic Getters and Setters

Properties in entities are typically `private` or `protected`, and access is controlled through getter and setter methods. This is the standard pattern:

```php
class Post
{
    private string $title;
    private string $body;
    private bool $published = false;

    public function getTitle(): string
    {
        return $this->title;
    }

    public function setTitle(string $title): void
    {
        $this->title = $title;
    }

    public function getBody(): string
    {
        return $this->body;
    }

    public function setBody(string $body): void
    {
        $this->body = $body;
    }

    public function isPublished(): bool
    {
        return $this->published;
    }

    public function setPublished(bool $published): void
    {
        $this->published = $published;
    }
}
```

Note that boolean properties conventionally use `is` or `has` as the getter prefix (`isPublished`, `hasImages`, `isActive`) rather than `get`.

### Getters with Logic

Getters do not have to be simple property returns. They can contain logic:

```php
class User
{
    private string $firstName;
    private string $lastName;
    private \DateTimeImmutable $birthDate;

    public function getFullName(): string
    {
        return $this->firstName . ' ' . $this->lastName;
    }

    public function getAge(): int
    {
        return (int) $this->birthDate->diff(new \DateTimeImmutable())->y;
    }

    public function isAdult(): bool
    {
        return $this->getAge() >= 18;
    }
}
```

### Setters with Validation Logic

Setters can also enforce business rules:

```php
class Product
{
    private float $price;
    private int $stock;

    public function setPrice(float $price): void
    {
        if ($price < 0) {
            throw new \InvalidArgumentException('Price cannot be negative.');
        }
        $this->price = $price;
    }

    public function setStock(int $stock): void
    {
        if ($stock < 0) {
            throw new \InvalidArgumentException('Stock cannot be negative.');
        }
        $this->stock = $stock;
    }
}
```

This is **domain validation** — the entity protects its own invariants at all times, regardless of whether the Symfony Validator is involved.

### Removing Setters (Immutable-style)

For properties that should never change after creation (like `createdAt` or `id`), simply do not provide a setter:

```php
class Order
{
    private \DateTimeImmutable $createdAt;

    public function __construct()
    {
        $this->createdAt = new \DateTimeImmutable();
    }

    public function getCreatedAt(): \DateTimeImmutable
    {
        return $this->createdAt;
    }

    // No setCreatedAt() — this value is set once and never changes
}
```

---

## 6. Fluent Interface (Method Chaining) {#fluent-interface}

A fluent interface lets you chain method calls on the same object. Setters return `$this` (or `static`) instead of `void`:

```php
class Article
{
    private string $title = '';
    private string $content = '';
    private bool $published = false;
    private ?string $category = null;

    public function setTitle(string $title): static
    {
        $this->title = $title;
        return $this;
    }

    public function setContent(string $content): static
    {
        $this->content = $content;
        return $this;
    }

    public function setPublished(bool $published): static
    {
        $this->published = $published;
        return $this;
    }

    public function setCategory(?string $category): static
    {
        $this->category = $category;
        return $this;
    }
}

// Usage with method chaining:
$article = (new Article())
    ->setTitle('My First Post')
    ->setContent('Hello World')
    ->setCategory('Technology')
    ->setPublished(true);
```

Using `static` instead of `self` as the return type is important for inheritance. `static` refers to the actual called class, while `self` refers to the class where the method is defined.

### When to Use Fluent Interface

Fluent interfaces are common in Symfony entities generated by `make:entity`. They work well for building objects and are readable. The tradeoff is that void setters (`void`) are clearer about what the method does, while fluent setters (`static`) prioritize convenience. Either style is valid.

---

## 7. PHP 8 Features Used in Entities {#php8-features}

### Constructor Property Promotion

PHP 8.0 introduced constructor property promotion, which collapses property declaration, constructor parameter, and assignment into one line. This is very useful for simple entities and especially for DTOs:

```php
// Before PHP 8 (verbose):
class Point
{
    private float $x;
    private float $y;

    public function __construct(float $x, float $y)
    {
        $this->x = $x;
        $this->y = $y;
    }
}

// With PHP 8 constructor promotion (concise):
class Point
{
    public function __construct(
        private float $x,
        private float $y,
    ) {}

    public function getX(): float { return $this->x; }
    public function getY(): float { return $this->y; }
}
```

Promotion can also be combined with attributes:

```php
use Symfony\Component\Validator\Constraints as Assert;
use Doctrine\ORM\Mapping as ORM;

class User
{
    public function __construct(
        #[Assert\NotBlank]
        #[Assert\Email]
        #[ORM\Column]
        private string $email,

        #[Assert\NotBlank]
        #[Assert\Length(min: 2)]
        #[ORM\Column]
        private string $username,
    ) {}
}
```

### Readonly Properties (PHP 8.1)

Readonly properties can only be initialized once. They cannot be modified after assignment:

```php
class UserId
{
    public function __construct(
        public readonly int $value
    ) {}
}

$id = new UserId(42);
echo $id->value; // 42
$id->value = 99; // Error: Cannot modify readonly property
```

This is ideal for identity fields and value objects where immutability is desired.

### Enums as Property Types (PHP 8.1)

PHP 8.1 backed enums work well as entity property types, replacing string or integer constants:

```php
enum OrderStatus: string
{
    case Draft    = 'draft';
    case Pending  = 'pending';
    case Paid     = 'paid';
    case Shipped  = 'shipped';
    case Cancelled = 'cancelled';
}

class Order
{
    private OrderStatus $status = OrderStatus::Draft;

    public function getStatus(): OrderStatus
    {
        return $this->status;
    }

    public function setStatus(OrderStatus $status): void
    {
        $this->status = $status;
    }

    public function cancel(): void
    {
        if ($this->status === OrderStatus::Shipped) {
            throw new \LogicException('Cannot cancel a shipped order.');
        }
        $this->status = OrderStatus::Cancelled;
    }
}
```

Doctrine 2.11+ supports mapping backed enums directly to database columns.

### Union Types

PHP 8.0 union types allow a property to accept multiple types:

```php
class Notification
{
    private int|string $recipient; // could be a user ID or an email address
}
```

Union types are used sparingly in entities because they can complicate type safety, but they are valid PHP.

### Nullsafe Operator in Getters

The nullsafe operator `?->` is useful when chaining calls on nullable related objects:

```php
$city = $user->getAddress()?->getCity()?->getName();
// Returns null at any point in the chain if a value is null,
// rather than throwing a "call on null" error.
```

---

## 8. Nullable Properties and Optional Fields {#nullable-properties}

In entities, some properties are required (must always have a value), and some are optional (may be null).

### Nullable Type Hints

Mark a property as nullable with `?` before the type:

```php
class UserProfile
{
    private string $username;           // Required — always has a value
    private ?string $bio = null;        // Optional — may be null
    private ?string $website = null;    // Optional — may be null
    private ?\DateTimeImmutable $dateOfBirth = null; // Optional
}
```

### Difference Between Null and Empty

Be deliberate about the difference:

- `null` means "this value was never provided or does not apply"
- `''` (empty string) means "this value was provided but is empty"

For optional text fields, `null` is usually the right default, not an empty string. This way, you can distinguish between "user did not provide a bio" and "user explicitly cleared their bio."

### Optional via Setter Return Value

A common pattern is to make the getter return a default when the value is null:

```php
class Product
{
    private ?string $description = null;

    public function getDescription(): string
    {
        return $this->description ?? 'No description available.';
    }

    public function getRawDescription(): ?string
    {
        return $this->description;
    }

    public function hasDescription(): bool
    {
        return $this->description !== null;
    }
}
```

---

## 9. Default Values {#default-values}

Default values ensure the entity starts in a valid and meaningful state.

### Property-Level Defaults

```php
class BlogPost
{
    private bool $published = false;
    private int $viewCount = 0;
    private array $tags = [];
    private string $status = 'draft';
    private \DateTimeImmutable $createdAt;

    public function __construct()
    {
        $this->createdAt = new \DateTimeImmutable();
    }
}
```

Objects (like `\DateTimeImmutable`) cannot be set as inline property defaults because property defaults must be constant expressions. They are initialized in the constructor instead.

### Enum Defaults

```php
enum PostStatus: string
{
    case Draft     = 'draft';
    case Published = 'published';
    case Archived  = 'archived';
}

class Post
{
    private PostStatus $status = PostStatus::Draft;
}
```

---

## 10. Read-Only Properties {#readonly-properties}

Some entity properties should be set once and never changed. There are two ways to achieve this.

### No Setter (Conventional Approach)

The simplest approach: declare the property, initialize it in the constructor, provide only a getter:

```php
class Event
{
    private \DateTimeImmutable $createdAt;
    private string $createdBy;

    public function __construct(string $createdBy)
    {
        $this->createdAt = new \DateTimeImmutable();
        $this->createdBy = $createdBy;
    }

    public function getCreatedAt(): \DateTimeImmutable
    {
        return $this->createdAt;
    }

    public function getCreatedBy(): string
    {
        return $this->createdBy;
    }

    // No setCreatedAt(), no setCreatedBy()
}
```

### PHP 8.1 Readonly Properties

With PHP 8.1, you can use the `readonly` modifier:

```php
class Category
{
    public readonly \DateTimeImmutable $createdAt;

    public function __construct(
        public readonly string $name,
    ) {
        $this->createdAt = new \DateTimeImmutable();
    }
}
```

A `readonly` property can only be written once. Any further attempt to set it throws an error. Note that `readonly` properties cannot have default values (except for constructor promotion).

### Readonly Classes (PHP 8.2)

PHP 8.2 introduced readonly classes, where all properties are implicitly readonly:

```php
readonly class Money
{
    public function __construct(
        public int $amount,
        public string $currency,
    ) {}
}
```

Readonly classes are excellent for value objects. They cannot be modified after creation.

---

## 11. Value Objects {#value-objects}

A value object is a small, immutable object that represents a concept defined entirely by its value. Unlike entities, value objects have no identity. Two `Money` objects both holding `100 EUR` are equal and interchangeable.

Value objects are not entities — they do not have an ID, they are not stored in their own table. Instead, they are embedded inside an entity.

### Example: Money Value Object

```php
final class Money
{
    public function __construct(
        private readonly int $amount,    // stored in cents to avoid float precision issues
        private readonly string $currency
    ) {
        if ($amount < 0) {
            throw new \InvalidArgumentException('Amount cannot be negative.');
        }

        if (strlen($currency) !== 3) {
            throw new \InvalidArgumentException('Currency must be a 3-letter ISO code.');
        }
    }

    public function getAmount(): int
    {
        return $this->amount;
    }

    public function getCurrency(): string
    {
        return $this->currency;
    }

    public function getFormattedAmount(): string
    {
        return number_format($this->amount / 100, 2) . ' ' . $this->currency;
    }

    public function equals(self $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }

    public function add(self $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException('Cannot add different currencies.');
        }
        return new self($this->amount + $other->amount, $this->currency);
    }
}
```

Usage inside an entity:

```php
class Order
{
    private Money $totalAmount;

    public function __construct(Money $totalAmount)
    {
        $this->totalAmount = $totalAmount;
    }

    public function getTotalAmount(): Money
    {
        return $this->totalAmount;
    }
}

$order = new Order(new Money(4999, 'EUR'));
echo $order->getTotalAmount()->getFormattedAmount(); // "49.99 EUR"
```

### Example: Email Value Object

```php
final class Email
{
    private string $value;

    public function __construct(string $value)
    {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException(sprintf('"%s" is not a valid email.', $value));
        }
        $this->value = strtolower($value);
    }

    public function getValue(): string
    {
        return $this->value;
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }

    public function __toString(): string
    {
        return $this->value;
    }
}
```

### Characteristics of a Value Object

- It is immutable — once created, it cannot be changed
- Equality is based on value, not identity
- It is small and focused on one concept
- It self-validates in the constructor
- It is usually declared `final` to prevent extension
- Operations that "modify" it return a new instance instead of mutating the current one

---

## 12. Embeddables {#embeddables}

An embeddable is Doctrine's way of mapping a value object to columns inside the owning entity's table. Instead of the `Address` having its own table with a foreign key, its fields are stored directly in the `User` table as `address_street`, `address_city`, etc.

From a pure PHP perspective (without thinking about persistence), an embeddable is just a regular PHP class used as a property type in an entity:

```php
class Address
{
    private string $street;
    private string $city;
    private string $postalCode;
    private string $countryCode;

    public function __construct(
        string $street,
        string $city,
        string $postalCode,
        string $countryCode
    ) {
        $this->street      = $street;
        $this->city        = $city;
        $this->postalCode  = $postalCode;
        $this->countryCode = $countryCode;
    }

    public function getStreet(): string    { return $this->street; }
    public function getCity(): string      { return $this->city; }
    public function getPostalCode(): string { return $this->postalCode; }
    public function getCountryCode(): string { return $this->countryCode; }

    public function getFullAddress(): string
    {
        return sprintf('%s, %s %s, %s',
            $this->street,
            $this->postalCode,
            $this->city,
            $this->countryCode
        );
    }
}
```

Used in an entity:

```php
class User
{
    private ?int $id = null;
    private string $name;
    private ?Address $homeAddress = null;

    public function getHomeAddress(): ?Address
    {
        return $this->homeAddress;
    }

    public function setHomeAddress(?Address $address): void
    {
        $this->homeAddress = $address;
    }
}
```

The class `Address` here is just a regular PHP class. When Doctrine maps it with `#[ORM\Embeddable]`, it takes each property of `Address` and stores it as a column in the `user` table, prefixed by default (e.g., `home_address_street`, `home_address_city`).

### Embeddable vs Separate Entity

Use an embeddable (value object) when:

- The object has no identity of its own
- It only makes sense in the context of its owning entity
- You do not need to query it independently
- It should travel with the entity

Use a separate entity (with its own table) when:

- It has its own lifecycle
- Multiple entities share the same instance
- You need to query it directly

---

## 13. Entity Relationships (Structure Overview) {#relationships}

When two entities are related to each other, you describe that relationship in the entity classes. The relationship itself is just a property that holds either a single related object or a collection of related objects. Doctrine handles the database side (foreign keys, join tables), but the PHP structure is straightforward.

There are four types of relationships:

### ManyToOne / OneToMany (The Most Common)

A `Comment` belongs to one `Post`. A `Post` has many `Comment` objects. This is ManyToOne from the `Comment` side and OneToMany from the `Post` side.

```php
// Comment.php — the "many" side (owns the relationship)
class Comment
{
    private ?int $id = null;
    private string $content;
    private Post $post; // reference to a single Post

    public function getPost(): Post
    {
        return $this->post;
    }

    public function setPost(Post $post): void
    {
        $this->post = $post;
    }
}

// Post.php — the "one" side (inverse side)
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;

class Post
{
    private ?int $id = null;
    private string $title;
    private Collection $comments; // collection of Comment objects

    public function __construct()
    {
        $this->comments = new ArrayCollection();
    }

    public function getComments(): Collection
    {
        return $this->comments;
    }

    public function addComment(Comment $comment): void
    {
        if (!$this->comments->contains($comment)) {
            $this->comments->add($comment);
            $comment->setPost($this);
        }
    }

    public function removeComment(Comment $comment): void
    {
        $this->comments->removeElement($comment);
    }
}
```

The important PHP concept here is that the `Post` holds a `Collection` of `Comment` objects, and the `Comment` holds a direct reference to a `Post`. The `addComment` method keeps both sides synchronized.

### OneToOne

A `User` has one `UserProfile`. A `UserProfile` belongs to one `User`.

```php
class User
{
    private ?UserProfile $profile = null;

    public function getProfile(): ?UserProfile
    {
        return $this->profile;
    }

    public function setProfile(?UserProfile $profile): void
    {
        $this->profile = $profile;
    }
}

class UserProfile
{
    private User $user;
    private string $bio = '';

    public function getUser(): User
    {
        return $this->user;
    }
}
```

### ManyToMany

A `Student` can enroll in many `Course` objects. A `Course` can have many `Student` objects.

```php
class Student
{
    private Collection $courses;

    public function __construct()
    {
        $this->courses = new ArrayCollection();
    }

    public function getCourses(): Collection
    {
        return $this->courses;
    }

    public function enroll(Course $course): void
    {
        if (!$this->courses->contains($course)) {
            $this->courses->add($course);
            $course->addStudent($this);
        }
    }
}

class Course
{
    private Collection $students;

    public function __construct()
    {
        $this->students = new ArrayCollection();
    }

    public function getStudents(): Collection
    {
        return $this->students;
    }

    public function addStudent(Student $student): void
    {
        if (!$this->students->contains($student)) {
            $this->students->add($student);
        }
    }
}
```

### Key Principle: Bidirectional Synchronization

When you have a bidirectional relationship, you are responsible for keeping both sides in sync. The `addComment` method sets the post on the comment. The `enroll` method adds the student to the course's collection. This is pure PHP responsibility — the entity manages its own state.

---

## 14. Collections in Entities {#collections}

When an entity holds multiple related objects, you use a `Collection`. Doctrine provides the `Doctrine\Common\Collections\Collection` interface and `ArrayCollection` as the default in-memory implementation.

### Why Not Just Use Arrays?

You could use a plain PHP array, but `Collection` provides a richer interface:

```php
use Doctrine\Common\Collections\Collection;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Criteria;
```

`Collection` offers methods like `filter()`, `map()`, `contains()`, `first()`, `last()`, `slice()`, and `matching()` (for Doctrine expressions). When Doctrine loads the entity from the database, it replaces your `ArrayCollection` with a lazy-loading proxy that only fetches related data when you first access the collection.

### Initializing Collections in the Constructor

Always initialize collections in the constructor. This ensures the property is never null, even before Doctrine has loaded anything:

```php
class Category
{
    private Collection $products;
    private Collection $subcategories;

    public function __construct()
    {
        $this->products      = new ArrayCollection();
        $this->subcategories = new ArrayCollection();
    }
}
```

### Filtering Collections

```php
class Order
{
    private Collection $items;

    public function __construct()
    {
        $this->items = new ArrayCollection();
    }

    public function getItems(): Collection
    {
        return $this->items;
    }

    public function getPhysicalItems(): Collection
    {
        return $this->items->filter(
            fn(OrderItem $item) => $item->getType() === 'physical'
        );
    }

    public function getTotalQuantity(): int
    {
        return $this->items->reduce(
            fn(int $carry, OrderItem $item) => $carry + $item->getQuantity(),
            0
        );
    }
}
```

### The `contains()` Pattern

Always check before adding to avoid duplicates:

```php
public function addTag(Tag $tag): void
{
    if (!$this->tags->contains($tag)) {
        $this->tags->add($tag);
    }
}
```

---

## 15. Entity Lifecycle and State {#lifecycle}

### The Four States of an Entity (Doctrine Context)

Even though this guide avoids deep Doctrine details, understanding these states helps you understand the entity's journey:

- **New (Transient)**: The entity was just created with `new`. Doctrine does not know about it.
- **Managed**: The entity is tracked by Doctrine's Unit of Work. Any changes will be detected and persisted on flush.
- **Detached**: The entity was once managed but is no longer tracked (e.g., after serialization or after the EntityManager was cleared).
- **Removed**: The entity is scheduled for deletion from the database.

From a pure PHP perspective, what matters is:

- Before persistence: `id` is `null`
- After persistence: `id` is populated

### Checking if an Entity is New

```php
public function isNew(): bool
{
    return $this->id === null;
}
```

### Lifecycle Callbacks

Doctrine provides lifecycle hooks that you can define directly on the entity. These are methods that Doctrine calls automatically at certain points. They are defined in the entity class but orchestrated by Doctrine:

```php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\HasLifecycleCallbacks]
class Article
{
    private \DateTimeImmutable $createdAt;
    private \DateTimeImmutable $updatedAt;

    #[ORM\PrePersist]
    public function onPrePersist(): void
    {
        $this->createdAt = new \DateTimeImmutable();
        $this->updatedAt = new \DateTimeImmutable();
    }

    #[ORM\PreUpdate]
    public function onPreUpdate(): void
    {
        $this->updatedAt = new \DateTimeImmutable();
    }
}
```

The key lifecycle callbacks are:

- `PrePersist` — called before the entity is first inserted
- `PostPersist` — called after the entity is first inserted (ID is available here)
- `PreUpdate` — called before an existing entity is updated
- `PostUpdate` — called after an existing entity is updated
- `PreRemove` — called before the entity is deleted
- `PostRemove` — called after the entity is deleted
- `PostLoad` — called after the entity is loaded from the database

---

## 16. DTOs vs Entities {#dtos-vs-entities}

This distinction is crucial in Symfony development. People often confuse DTOs and entities or use them interchangeably. They serve different purposes.

### What Is an Entity?

An entity represents a domain concept with identity and business behavior. It is the "true" state of your domain, usually persisted in a database. It enforces invariants and contains business logic.

```php
class User
{
    private ?int $id = null;
    private string $email;
    private string $hashedPassword;
    private \DateTimeImmutable $createdAt;

    public function __construct(string $email, string $hashedPassword)
    {
        $this->email          = $email;
        $this->hashedPassword = $hashedPassword;
        $this->createdAt      = new \DateTimeImmutable();
    }

    public function changeEmail(string $newEmail): void
    {
        // could trigger a domain event, validation, etc.
        $this->email = $newEmail;
    }
}
```

### What Is a DTO?

A **Data Transfer Object** is a simple container for data with no business logic. It carries data between layers (e.g., from an HTTP request to a service, or from a service to a template). It has no identity, no behavior, and is usually short-lived.

```php
class CreateUserRequest
{
    public string $email = '';
    public string $password = '';
    public string $confirmPassword = '';
}
```

### When to Use Each

Use an **entity** when:

- The data represents a persisted domain concept
- The object has a lifecycle (create, update, delete)
- Business rules need to be enforced on the data

Use a **DTO** when:

- You are passing data from a form or API request
- You want to decouple the input format from the entity structure
- You have computed or aggregated data for display purposes
- You are returning data from an API and do not want to expose the entity directly

### Mapping a DTO to an Entity

The service layer typically handles the translation:

```php
class UserService
{
    public function createUser(CreateUserRequest $dto): User
    {
        if ($dto->password !== $dto->confirmPassword) {
            throw new \InvalidArgumentException('Passwords do not match.');
        }

        $hashedPassword = password_hash($dto->password, PASSWORD_BCRYPT);

        return new User($dto->email, $hashedPassword);
    }
}
```

### The Read DTO (Response DTO)

You often also create DTOs for the output side, to avoid exposing sensitive entity fields:

```php
class UserResponse
{
    public int $id;
    public string $email;
    public string $createdAt;

    public static function fromEntity(User $user): self
    {
        $response            = new self();
        $response->id        = $user->getId();
        $response->email     = $user->getEmail();
        $response->createdAt = $user->getCreatedAt()->format('Y-m-d H:i:s');
        return $response;
    }
}
```

This pattern ensures your API response never accidentally exposes the hashed password or other internal fields.

---

## 17. Entity Design Patterns and Best Practices {#best-practices}

### Keep Properties Private

Always declare entity properties as `private`. This forces consumers of the entity to go through the getter/setter API, giving you full control over what can be read and changed. `protected` is acceptable if you have entity inheritance. `public` is almost never appropriate.

### Rich Domain Model vs Anemic Domain Model

An **anemic domain model** is an entity that is only a data bag — all its properties are exposed via getters and setters with no logic. All business logic lives in services.

A **rich domain model** is an entity that contains the business logic relevant to itself. It does not just store data — it understands its own rules.

Anemic (avoid):

```php
class Order
{
    private string $status;
    public function getStatus(): string { return $this->status; }
    public function setStatus(string $status): void { $this->status = $status; }
}

// Business logic lives in a service — bad
class OrderService
{
    public function ship(Order $order): void
    {
        if ($order->getStatus() !== 'paid') {
            throw new \LogicException('Cannot ship unpaid order.');
        }
        $order->setStatus('shipped');
    }
}
```

Rich (prefer):

```php
class Order
{
    private string $status = 'draft';

    public function pay(): void
    {
        if ($this->status !== 'pending') {
            throw new \LogicException('Only pending orders can be paid.');
        }
        $this->status = 'paid';
    }

    public function ship(): void
    {
        if ($this->status !== 'paid') {
            throw new \LogicException('Cannot ship an unpaid order.');
        }
        $this->status = 'shipped';
    }

    public function isPaid(): bool { return $this->status === 'paid'; }
    public function isShipped(): bool { return $this->status === 'shipped'; }
}
```

The rich model is self-protecting. An `Order` can only be shipped if it has been paid. The entity enforces this, not some external service.

### Use Immutable DateTimeImmutable over DateTime

Always use `\DateTimeImmutable` instead of `\DateTime` for date properties in entities. `DateTime` is mutable — if you call `$date->modify('+1 day')` it changes the object itself, which can cause subtle bugs. `DateTimeImmutable` always returns a new object.

```php
// Bad — mutable, can be modified from outside
private \DateTime $createdAt;

// Good — immutable
private \DateTimeImmutable $createdAt;
```

### Protect Collections

Return collections as interface types or, better, as unmodifiable views:

```php
use Doctrine\Common\Collections\Collection;
use Doctrine\Common\Collections\ReadableCollection;

public function getTags(): Collection
{
    return $this->tags;
}
```

If you want to truly prevent outside modification, return an `array` snapshot instead:

```php
public function getTags(): array
{
    return $this->tags->toArray();
}
```

### Use Enums for Statuses and Fixed Lists

Do not use magic strings for statuses, types, or any fixed list of values. Use PHP 8.1 backed enums:

```php
// Bad
$product->setStatus('active'); // typos are silent bugs

// Good
$product->setStatus(ProductStatus::Active); // type-safe, IDE-supported
```

### Avoid Logic in Setters That Belongs Elsewhere

Setters should be simple. Complex business transitions should be explicit methods:

```php
// Bad — setter does too much
public function setStatus(string $status): void
{
    if ($status === 'published' && $this->publishedAt === null) {
        $this->publishedAt = new \DateTimeImmutable();
    }
    $this->status = $status;
}

// Good — explicit domain method
public function publish(): void
{
    if ($this->status === 'published') {
        return;
    }
    $this->status      = 'published';
    $this->publishedAt = new \DateTimeImmutable();
}
```

---

## 18. Common Mistakes {#common-mistakes}

### Mistake 1: Using `DateTime` Instead of `DateTimeImmutable`

As explained above, always prefer `DateTimeImmutable`. The `DateTime` class is mutable and its mutation affects the stored value.

### Mistake 2: Public Properties

Making entity properties `public` bypasses encapsulation and removes your ability to enforce any rules on access or mutation.

### Mistake 3: Forgetting to Initialize Collections

If you forget to initialize a collection in the constructor and then try to call `add()` on it, PHP will throw an error because the property is `null`, not an `ArrayCollection`.

```php
// This will crash if $this->tags is null
public function addTag(Tag $tag): void
{
    $this->tags->add($tag); // Error if not initialized
}

// Always initialize in the constructor
public function __construct()
{
    $this->tags = new ArrayCollection();
}
```

### Mistake 4: Bidirectional Relationship Without Synchronization

If you have a bidirectional relationship and only set one side, the other side will not be updated until the entity is reloaded from the database:

```php
// Bad — only sets one side
$comment->setPost($post);
// $post->getComments() does not contain $comment until reload

// Good — set both sides
$comment->setPost($post);
$post->addComment($comment);
// or let addComment() handle both sides internally
```

### Mistake 5: Returning null from Collections

Never return `null` from a collection getter. If the entity has not been persisted yet and the collection is empty, return an empty `ArrayCollection` or an empty array.

### Mistake 6: Putting Infrastructure Logic in Entities

Entities should not know about HTTP, sessions, the filesystem, email sending, or any infrastructure concerns. They are domain objects. Infrastructure belongs in services.

### Mistake 7: Comparing Entities with `===`

Two entity objects are the same entity if they have the same ID. Using `===` compares object identity (same PHP instance), not domain identity:

```php
// Bad — might return false even for the same user
if ($userA === $userB) { ... }

// Good
if ($userA->getId() !== null && $userA->getId() === $userB->getId()) { ... }
```

Or implement `equals()`:

```php
public function equals(self $other): bool
{
    return $this->id !== null && $this->id === $other->id;
}
```

---

## Quick Reference Summary

| Concept             | Key Point                                                           |
| ------------------- | ------------------------------------------------------------------- |
| Entity              | Plain PHP class with an identity (id), no required base class       |
| Property visibility | Always `private`                                                    |
| ID type             | `?int` (nullable until persisted)                                   |
| Dates               | Always use `DateTimeImmutable`                                      |
| Collections         | Always initialize with `ArrayCollection` in constructor             |
| Statuses/Types      | Use PHP 8.1 backed enums                                            |
| Optional fields     | Use `?Type = null`, not empty strings                               |
| Value Object        | No ID, immutable, equality by value, self-validating                |
| Embeddable          | Value object mapped to owning entity's columns                      |
| DTO                 | No behavior, carries data between layers, short-lived               |
| Rich model          | Entity contains its own business logic and invariants               |
| Lifecycle callbacks | Defined in entity, called by Doctrine at persist/load/update/delete |
| Bidirectional sync  | Always update both sides of a relationship manually                 |

---

## Closing Thoughts

An entity in Symfony is just a PHP class, and that simplicity is its strength. The design decisions you make — which properties are required, which are optional, where business logic lives, how you model relationships, when to use value objects — all of these are PHP and domain design decisions, completely independent of any framework or database technology.

Mastering entities at the pure PHP level first gives you a solid foundation for when you add Doctrine mapping on top. The mapping layer simply tells Doctrine how to store and retrieve what is already a well-designed PHP object.
