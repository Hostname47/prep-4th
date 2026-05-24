# Symfony 7.4 Validation - Complete Guide

## Table of Contents

1. [Introduction to Validation](#introduction)
2. [Installation and Setup](#installation)
3. [Basic Validation Concepts](#basic-concepts)
4. [Defining Constraints](#defining-constraints)
5. [Built-in Constraints Reference](#built-in-constraints)
6. [Validating in Controllers](#validating-in-controllers)
7. [Validation Groups](#validation-groups)
8. [Cascading Validation](#cascading-validation)
9. [Custom Constraints](#custom-constraints)
10. [Sequenced Constraints](#sequenced-constraints)
11. [Validation in Forms](#validation-in-forms)
12. [Validation in APIs (Serializer + Validator)](#validation-in-apis)
13. [Translating Violation Messages](#translating-messages)
14. [Advanced Scenarios](#advanced-scenarios)
15. [Testing Validation](#testing-validation)

---

## 1. Introduction to Validation {#introduction}

Validation in Symfony is the process of checking that data conforms to a set of rules before using it. Symfony's Validator component is built on top of the open-source `symfony/validator` package and implements the JSR-303 Bean Validation specification adapted for PHP.

The core idea is simple: you attach **constraints** to your classes, properties, or methods. When you call the validator, it loops through those constraints, runs each one, and collects any violations. A violation is a structured object describing what went wrong, which property failed, and what the expected value should be.

Validation is completely decoupled from the persistence layer. You can validate any PHP object, DTO, form data model, or API request body regardless of whether you use Doctrine, another ORM, or nothing at all.

---

## 2. Installation and Setup {#installation}

### Installing the Validator Component

In a full Symfony application created with `symfony new`, the validator is usually included via the `symfony/validator` package. If it is not present, install it:

```bash
composer require symfony/validator
```

If you also want annotation/attribute support (which is the modern approach in Symfony 7.x), no extra package is needed since attributes are native PHP 8.

If you want YAML or XML constraint configuration, the validator reads them automatically when placed in the correct locations.

### Enabling Validation

In `config/packages/framework.yaml`, validation is enabled by default:

```yaml
framework:
  validation:
    enabled: true
    email_validation_mode: html5
```

The `email_validation_mode` option controls how `#[Email]` works. Options are `html5`, `strict`, and `loose`.

---

## 3. Basic Validation Concepts {#basic-concepts}

### The Validator Service

The main entry point is the `Symfony\Component\Validator\Validator\ValidatorInterface`. You can inject it anywhere via dependency injection:

```php
use Symfony\Component\Validator\Validator\ValidatorInterface;

class SomeService
{
    public function __construct(private ValidatorInterface $validator) {}
}
```

### Validating an Object

```php
$violations = $this->validator->validate($object);

if (count($violations) > 0) {
    // there are errors
    foreach ($violations as $violation) {
        echo $violation->getPropertyPath() . ': ' . $violation->getMessage();
    }
}
```

### The ConstraintViolationList

`validate()` returns a `ConstraintViolationList`. Each item is a `ConstraintViolation` with the following useful methods:

- `getMessage()` - the human-readable error message
- `getPropertyPath()` - the path to the invalid property (e.g., `name`, `address.city`)
- `getInvalidValue()` - the value that failed validation
- `getCode()` - a machine-readable error code constant defined on the constraint class
- `getConstraint()` - the actual constraint object that triggered the violation

---

## 4. Defining Constraints {#defining-constraints}

Symfony 7.x recommends using **PHP attributes** to define constraints. You can also use YAML, XML, or PHP configuration files.

### Using PHP Attributes (Recommended)

```php
namespace App\Entity;

use Symfony\Component\Validator\Constraints as Assert;

class User
{
    #[Assert\NotBlank(message: 'The name cannot be empty.')]
    #[Assert\Length(min: 2, max: 100)]
    public string $name = '';

    #[Assert\NotBlank]
    #[Assert\Email(mode: 'html5')]
    public string $email = '';

    #[Assert\PositiveOrZero]
    public int $age = 0;
}
```

### Class-level vs Property-level Constraints

Most constraints apply to individual properties. Some constraints apply at the class level, which lets them access multiple properties at once. Class-level constraints are placed above the class definition:

```php
#[Assert\Callback('validateDates')]
class Event
{
    public \DateTimeInterface $startDate;
    public \DateTimeInterface $endDate;

    public function validateDates(ExecutionContextInterface $context): void
    {
        if ($this->startDate >= $this->endDate) {
            $context->buildViolation('End date must be after start date.')
                ->atPath('endDate')
                ->addViolation();
        }
    }
}
```

---

## 5. Built-in Constraints Reference {#built-in-constraints}

### Basic Constraints

**NotBlank** - Ensures the value is not empty (not null, not empty string, not empty array).

```php
#[Assert\NotBlank(message: 'This field is required.')]
public string $title;
```

**NotNull** - Ensures the value is not null (empty string would pass).

```php
#[Assert\NotNull]
public ?string $status;
```

**IsNull** - Ensures the value is null.

**IsTrue / IsFalse** - Ensures the value is true or false (useful on boolean properties or getter methods).

```php
#[Assert\IsTrue(message: 'You must agree to the terms.')]
public bool $agreedToTerms;
```

**Type** - Ensures the value is a specific PHP type.

```php
#[Assert\Type(type: 'integer')]
public mixed $count;

#[Assert\Type(type: \DateTimeInterface::class)]
public mixed $publishedAt;
```

---

### String Constraints

**Length** - Validates string length.

```php
#[Assert\Length(
    min: 3,
    max: 255,
    minMessage: 'Must be at least {{ limit }} characters.',
    maxMessage: 'Cannot be longer than {{ limit }} characters.'
)]
public string $username;
```

**Email** - Validates email format.

```php
#[Assert\Email(mode: 'html5')]
public string $email;
```

Modes: `loose`, `html5` (default), `strict` (uses egulias/email-validator).

**Url** - Validates a URL.

```php
#[Assert\Url(protocols: ['http', 'https'])]
public string $website;
```

**Regex** - Validates against a regular expression.

```php
#[Assert\Regex(
    pattern: '/^\+?[0-9\s\-]{7,20}$/',
    message: 'Invalid phone number format.'
)]
public string $phone;
```

**NoSuspiciousCharacters** - Checks for potentially confusing Unicode characters (spoofing protection), introduced in Symfony 6.3.

```php
#[Assert\NoSuspiciousCharacters]
public string $username;
```

**Hostname** - Validates a hostname.

**Ip** - Validates an IP address (IPv4, IPv6, or both).

```php
#[Assert\Ip(version: '4')]
public string $ipAddress;
```

---

### Number Constraints

**Positive / PositiveOrZero** - Ensures the value is greater than zero (or zero and above).

```php
#[Assert\Positive(message: 'Price must be positive.')]
public float $price;
```

**Negative / NegativeOrZero**

**Range** - Validates that a number (or date) is within a range.

```php
#[Assert\Range(min: 1, max: 150)]
public int $age;

#[Assert\Range(
    min: 'today',
    max: '+1 year',
    notInRangeMessage: 'Date must be between today and one year from now.'
)]
public \DateTimeInterface $eventDate;
```

**DivisibleBy** - Ensures the value is divisible by a given number.

```php
#[Assert\DivisibleBy(value: 5)]
public int $quantity;
```

**LessThan / LessThanOrEqual / GreaterThan / GreaterThanOrEqual** - Comparisons against a fixed value or against another property.

```php
#[Assert\GreaterThan(value: 0)]
public float $discount;

#[Assert\GreaterThan(propertyPath: 'startDate')]
public \DateTimeInterface $endDate;
```

---

### Date and Time Constraints

**Date** - Validates a date string in `YYYY-MM-DD` format.

```php
#[Assert\Date]
public string $birthdate;
```

**DateTime** - Validates a datetime string.

```php
#[Assert\DateTime(format: 'Y-m-d H:i:s')]
public string $scheduledAt;
```

**Time** - Validates a time string.

**Timezone** - Validates a timezone identifier.

```php
#[Assert\Timezone]
public string $userTimezone;
```

---

### Collection / Array Constraints

**Count** - Validates the count of an array or collection.

```php
#[Assert\Count(min: 1, max: 10, minMessage: 'You must add at least one item.')]
public array $tags = [];
```

**UniqueEntity** (Doctrine-specific) - Ensures a field is unique in the database. This comes from `symfony/doctrine-bridge`, not the core Validator.

```php
use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

#[UniqueEntity(fields: ['email'], message: 'This email is already in use.')]
class User { ... }
```

**Unique** - Validates that all elements in an array are unique.

```php
#[Assert\Unique]
public array $selectedColors;
```

**Choice** - Validates that the value is one of a set of allowed values.

```php
#[Assert\Choice(choices: ['admin', 'editor', 'viewer'], message: 'Invalid role.')]
public string $role;

// For multiple choices:
#[Assert\Choice(choices: ['php', 'javascript', 'python'], multiple: true)]
public array $languages;
```

**Collection** - Validates each key of an array with specific constraints per key.

```php
#[Assert\Collection([
    'name'  => [new Assert\NotBlank(), new Assert\Length(min: 2)],
    'email' => [new Assert\NotBlank(), new Assert\Email()],
    'age'   => new Assert\Range(min: 18, max: 99),
])]
public array $profileData;
```

**All** - Applies constraints to every element in an array.

```php
#[Assert\All([
    new Assert\NotBlank(),
    new Assert\Length(max: 50),
])]
public array $tags;
```

---

### File and Image Constraints

**File** - Validates uploaded files.

```php
use Symfony\Component\HttpFoundation\File\UploadedFile;

#[Assert\File(
    maxSize: '2M',
    mimeTypes: ['application/pdf', 'image/jpeg'],
    mimeTypesMessage: 'Only PDF and JPEG files are allowed.'
)]
public ?UploadedFile $attachment = null;
```

**Image** - Validates image files with additional image-specific options.

```php
#[Assert\Image(
    maxWidth: 1920,
    maxHeight: 1080,
    minWidth: 100,
    minHeight: 100,
    maxSize: '5M',
    allowLandscape: true,
    allowPortrait: false
)]
public ?UploadedFile $photo = null;
```

---

### Financial and Other Constraints

**Iban** - Validates IBAN bank account numbers.

**Bic** - Validates BIC/SWIFT codes.

**Luhn** - Validates numbers using the Luhn algorithm (credit card numbers).

**Currency** - Validates a 3-letter ISO 4217 currency code.

**Locale** - Validates a locale string.

**Language** - Validates a language code.

**Country** - Validates a 2-letter ISO 3166-1 country code.

```php
#[Assert\Currency]
public string $currency;

#[Assert\Country]
public string $countryCode;
```

---

### Compound Constraints

**AtLeastOneOf** - Valid if at least one of the nested constraints passes. Useful for "or" logic.

```php
#[Assert\AtLeastOneOf([
    new Assert\Length(min: 10),
    new Assert\EqualTo('N/A'),
])]
public string $description;
```

**Sequentially** - Stops evaluating on the first violation. Useful to avoid cascading errors.

```php
#[Assert\Sequentially([
    new Assert\NotBlank(),
    new Assert\Length(min: 8),
    new Assert\Regex('/[A-Z]/'),
])]
public string $password;
```

**When** - Applies constraints conditionally based on an expression.

```php
#[Assert\When(
    expression: 'this.isCompany === true',
    constraints: [new Assert\NotBlank()],
    otherwise: [],
    message: 'Company name is required for companies.'
)]
public ?string $companyName = null;

public bool $isCompany = false;
```

---

### Expression Constraint

**Expression** - Uses Symfony's ExpressionLanguage to validate with a custom expression.

```php
#[Assert\Expression(
    expression: 'this.getPassword() === this.getConfirmPassword()',
    message: 'Passwords do not match.'
)]
class ChangePasswordRequest
{
    public string $password;
    public string $confirmPassword;

    public function getPassword(): string { return $this->password; }
    public function getConfirmPassword(): string { return $this->confirmPassword; }
}
```

---

## 6. Validating in Controllers {#validating-in-controllers}

### Manual Validation in a Controller

```php
namespace App\Controller;

use App\Entity\Product;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Validator\Validator\ValidatorInterface;

class ProductController extends AbstractController
{
    public function create(Request $request, ValidatorInterface $validator): JsonResponse
    {
        $product = new Product();
        $product->name  = $request->request->get('name', '');
        $product->price = (float) $request->request->get('price', 0);

        $violations = $validator->validate($product);

        if (count($violations) > 0) {
            $errors = [];
            foreach ($violations as $violation) {
                $errors[$violation->getPropertyPath()][] = $violation->getMessage();
            }
            return $this->json(['errors' => $errors], 422);
        }

        // Proceed to persist and return success
        return $this->json(['message' => 'Product created.'], 201);
    }
}
```

### Validating a Specific Property

Sometimes you only need to check one property without constructing a full object:

```php
$violations = $validator->validatePropertyValue(
    User::class,
    'email',
    'not-a-valid-email'
);
```

Or validate a property on an existing object:

```php
$violations = $validator->validateProperty($user, 'email');
```

---

## 7. Validation Groups {#validation-groups}

Validation groups let you run only a subset of constraints depending on context. For example, you might want stricter rules when creating a user versus updating one.

### Defining Groups on Constraints

```php
use Symfony\Component\Validator\Constraints as Assert;

class User
{
    #[Assert\NotBlank(groups: ['create', 'update'])]
    #[Assert\Length(min: 2, groups: ['create', 'update'])]
    public string $name = '';

    #[Assert\NotBlank(groups: ['create'])]
    #[Assert\Length(min: 8, groups: ['create'])]
    public string $password = '';

    #[Assert\NotBlank(groups: ['update'])]
    public ?string $bio = null;
}
```

A constraint with no `groups` option belongs to the `Default` group implicitly.

### Triggering Validation with Groups

```php
// Only run constraints in the 'create' group
$violations = $validator->validate($user, null, ['create']);

// Run multiple groups
$violations = $validator->validate($user, null, ['create', 'Default']);
```

### GroupSequence - Running Groups in Order

`GroupSequence` lets you run groups one by one and stop at the first group with violations. This prevents a flood of errors.

```php
use Symfony\Component\Validator\Constraints\GroupSequence;

#[Assert\GroupSequence(['User', 'StrongPassword'])]
class User
{
    #[Assert\NotBlank(groups: ['User'])]
    #[Assert\Length(min: 3, groups: ['User'])]
    public string $username = '';

    #[Assert\Length(min: 8, groups: ['StrongPassword'])]
    #[Assert\Regex(pattern: '/[A-Z]/', groups: ['StrongPassword'])]
    public string $password = '';
}

$violations = $validator->validate($user, null, new GroupSequence(['User', 'StrongPassword']));
```

In this example, if the `User` group has violations, the `StrongPassword` group is never evaluated.

### GroupSequenceProvider - Dynamic Groups

If you need groups to change based on the object's state, implement `GroupSequenceProviderInterface`:

```php
use Symfony\Component\Validator\GroupSequenceProviderInterface;

#[Assert\GroupSequence(['User', 'strictChecks'])]
class User implements GroupSequenceProviderInterface
{
    public bool $isPremium = false;

    public function getGroupSequence(): array|GroupSequence
    {
        $groups = ['User'];

        if ($this->isPremium) {
            $groups[] = 'Premium';
        }

        return $groups;
    }
}
```

Enable this in YAML if you are not using attributes:

```yaml
App\Entity\User:
  group_sequence_provider: true
```

---

## 8. Cascading Validation {#cascading-validation}

When a class contains another object as a property, Symfony will not automatically validate the nested object unless you tell it to. Use `#[Assert\Valid]` to trigger cascaded validation.

### Basic Cascading

```php
class Order
{
    #[Assert\NotBlank]
    public string $reference = '';

    #[Assert\Valid]
    public ?Address $shippingAddress = null;
}

class Address
{
    #[Assert\NotBlank]
    public string $street = '';

    #[Assert\NotBlank]
    #[Assert\Length(min: 2, max: 2)]
    public string $countryCode = '';
}
```

When you validate an `Order`, the validator will also validate the nested `Address`. Violation paths will reflect the nesting: `shippingAddress.street`.

### Cascading on Collections

If you have a collection of objects and want each one validated:

```php
class Order
{
    /**
     * @var OrderItem[]
     */
    #[Assert\Valid]
    #[Assert\Count(min: 1)]
    public array $items = [];
}

class OrderItem
{
    #[Assert\NotBlank]
    public string $productName = '';

    #[Assert\Positive]
    public int $quantity = 1;
}
```

---

## 9. Custom Constraints {#custom-constraints}

When built-in constraints are not enough, you can create your own. A custom constraint consists of two classes: the **constraint** class (which describes the error message and options) and the **validator** class (which contains the actual logic).

### Example: Validating a Slug

**The Constraint class:**

```php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;

#[\Attribute(\Attribute::TARGET_PROPERTY | \Attribute::TARGET_METHOD | \Attribute::IS_REPEATABLE)]
class Slug extends Constraint
{
    public string $message = 'The value "{{ value }}" is not a valid slug.';
}
```

**The Validator class:**

```php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;
use Symfony\Component\Validator\Exception\UnexpectedTypeException;

class SlugValidator extends ConstraintValidator
{
    public function validate(mixed $value, Constraint $constraint): void
    {
        if (!$constraint instanceof Slug) {
            throw new UnexpectedTypeException($constraint, Slug::class);
        }

        if (null === $value || '' === $value) {
            return; // Let NotBlank handle empty values
        }

        if (!preg_match('/^[a-z0-9]+(?:-[a-z0-9]+)*$/', $value)) {
            $this->context->buildViolation($constraint->message)
                ->setParameter('{{ value }}', $value)
                ->addViolation();
        }
    }
}
```

Symfony automatically resolves the validator class by convention: `SlugValidator` for `Slug`. If you want a non-conventional name, override `validatedBy()` in the constraint class.

**Usage:**

```php
use App\Validator\Slug;

class Article
{
    #[Slug]
    public string $slug = '';
}
```

---

### Example: Class-Level Custom Constraint (Password Match)

```php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;

#[\Attribute(\Attribute::TARGET_CLASS)]
class PasswordMatch extends Constraint
{
    public string $message = 'The passwords do not match.';

    public function getTargets(): string
    {
        return self::CLASS_CONSTRAINT;
    }
}
```

```php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;

class PasswordMatchValidator extends ConstraintValidator
{
    public function validate(mixed $value, Constraint $constraint): void
    {
        if (!$constraint instanceof PasswordMatch) {
            throw new UnexpectedTypeException($constraint, PasswordMatch::class);
        }

        if ($value->password !== $value->confirmPassword) {
            $this->context->buildViolation($constraint->message)
                ->atPath('confirmPassword')
                ->addViolation();
        }
    }
}
```

**Usage:**

```php
#[PasswordMatch]
class ChangePasswordRequest
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 8)]
    public string $password = '';

    #[Assert\NotBlank]
    public string $confirmPassword = '';
}
```

---

### Example: Service-Injected Custom Constraint

If your validator needs a service (like a database check for uniqueness):

```php
namespace App\Validator;

use App\Repository\UserRepository;
use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;

class UniqueUsernameValidator extends ConstraintValidator
{
    public function __construct(private UserRepository $userRepository) {}

    public function validate(mixed $value, Constraint $constraint): void
    {
        if (null === $value || '' === $value) {
            return;
        }

        if ($this->userRepository->findOneByUsername($value)) {
            $this->context->buildViolation($constraint->message)
                ->setParameter('{{ value }}', $value)
                ->addViolation();
        }
    }
}
```

Symfony's autowiring will automatically inject dependencies into the validator class, as long as it is registered as a service (which it is by default).

---

## 10. Sequenced Constraints {#sequenced-constraints}

The `Sequentially` compound constraint is extremely useful when you have a chain of constraints on a property and want to stop after the first failure. This avoids confusing, redundant errors.

### Without Sequentially

Without sequencing, every constraint is checked, and you might get multiple errors for the same field:

```php
// Could produce: "This value should not be blank." AND "This value is too short."
#[Assert\NotBlank]
#[Assert\Length(min: 5)]
public string $username = '';
```

### With Sequentially

```php
#[Assert\Sequentially([
    new Assert\NotBlank(),
    new Assert\Length(min: 5, max: 30),
    new Assert\Regex(
        pattern: '/^[a-zA-Z0-9_]+$/',
        message: 'Only letters, numbers, and underscores are allowed.'
    ),
])]
public string $username = '';
```

Now, if the username is blank, only one error is shown. The `Length` and `Regex` checks are skipped entirely.

---

## 11. Validation in Forms {#validation-in-forms}

Symfony Forms integrate deeply with the Validator component. When you call `$form->handleRequest($request)` and then `$form->isValid()`, validation runs automatically.

### Basic Form Validation

```php
// src/Form/RegistrationFormType.php
namespace App\Form;

use App\Entity\User;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\EmailType;
use Symfony\Component\Form\Extension\Core\Type\PasswordType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class RegistrationFormType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('name', TextType::class)
            ->add('email', EmailType::class)
            ->add('password', PasswordType::class);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => User::class,
        ]);
    }
}
```

In the controller:

```php
public function register(Request $request): Response
{
    $user = new User();
    $form = $this->createForm(RegistrationFormType::class, $user);
    $form->handleRequest($request);

    if ($form->isSubmitted() && $form->isValid()) {
        // $user is now populated and validated
        // persist, redirect, etc.
    }

    return $this->render('register.html.twig', ['form' => $form]);
}
```

### Displaying Errors in Twig

```twig
{{ form_start(form) }}

{{ form_errors(form) }}

<div>
    {{ form_label(form.name) }}
    {{ form_widget(form.name) }}
    {{ form_errors(form.name) }}
</div>

<div>
    {{ form_label(form.email) }}
    {{ form_widget(form.email) }}
    {{ form_errors(form.email) }}
</div>

<button type="submit">Register</button>

{{ form_end(form) }}
```

### Using Validation Groups with Forms

```php
public function configureOptions(OptionsResolver $resolver): void
{
    $resolver->setDefaults([
        'data_class'        => User::class,
        'validation_groups' => ['Default', 'create'],
    ]);
}
```

You can also make the groups dynamic using a closure:

```php
'validation_groups' => function (FormInterface $form): array {
    $user = $form->getData();
    if ($user->getId()) {
        return ['Default', 'update'];
    }
    return ['Default', 'create'];
},
```

### Form-Only Constraints (Not on the Entity)

Sometimes you want constraints that only apply during form submission but should not be part of the entity's class constraints. Add them directly on the form field:

```php
use Symfony\Component\Validator\Constraints as Assert;

$builder->add('terms', CheckboxType::class, [
    'constraints' => [
        new Assert\IsTrue(message: 'You must accept the terms of service.'),
    ],
    'mapped' => false,
]);
```

Setting `'mapped' => false` means this field does not map to any entity property.

---

## 12. Validation in APIs {#validation-in-apis}

### Validating Deserialized Objects

When building an API, you typically deserialize the request body into a DTO and then validate it.

**The DTO:**

```php
namespace App\Dto;

use Symfony\Component\Validator\Constraints as Assert;

class CreateProductRequest
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 2, max: 200)]
    public string $name = '';

    #[Assert\NotBlank]
    #[Assert\Positive]
    public float $price = 0.0;

    #[Assert\NotBlank]
    #[Assert\Choice(choices: ['physical', 'digital', 'service'])]
    public string $type = 'physical';
}
```

**The Controller:**

```php
use App\Dto\CreateProductRequest;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Serializer\SerializerInterface;
use Symfony\Component\Validator\Validator\ValidatorInterface;

public function create(
    Request $request,
    SerializerInterface $serializer,
    ValidatorInterface $validator
): JsonResponse {
    $dto = $serializer->deserialize(
        $request->getContent(),
        CreateProductRequest::class,
        'json'
    );

    $violations = $validator->validate($dto);

    if (count($violations) > 0) {
        $errors = [];
        foreach ($violations as $violation) {
            $errors[] = [
                'field'   => $violation->getPropertyPath(),
                'message' => $violation->getMessage(),
            ];
        }
        return $this->json(['errors' => $errors], 422);
    }

    // use $dto to create the product
    return $this->json(['status' => 'created'], 201);
}
```

### Using Symfony HttpKernel's #[MapRequestPayload]

Symfony 6.3+ introduced `#[MapRequestPayload]`, which automates deserialization and validation:

```php
use App\Dto\CreateProductRequest;
use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;

public function create(
    #[MapRequestPayload] CreateProductRequest $payload
): JsonResponse {
    // If validation fails, Symfony throws HttpException with 422 status automatically.
    // $payload is already validated when this code runs.

    return $this->json(['status' => 'created'], 201);
}
```

You can pass validation groups:

```php
#[MapRequestPayload(validationGroups: ['create'])]
CreateProductRequest $payload
```

---

## 13. Translating Violation Messages {#translating-messages}

Symfony's Validator component has built-in support for translating messages via the Translation component.

### Default Translation Files

Symfony ships with translations for all built-in constraints in the `validators` translation domain. They are located inside the Validator component itself. When you run:

```bash
composer require symfony/translation
```

Symfony will automatically use the locale from the request to translate standard error messages.

### Custom Translations

To translate your custom constraint messages, create a translation file in `translations/validators.en.yaml`:

```yaml
The value "{{ value }}" is not a valid slug.: 'The value "{{ value }}" is not a valid slug.'
```

And for French in `translations/validators.fr.yaml`:

```yaml
The value "{{ value }}" is not a valid slug.: 'La valeur "{{ value }}" n''est pas un identifiant valide.'
```

### Custom Message with Translation Keys

A better practice is to use translation keys as your message:

```php
#[\Attribute(\Attribute::TARGET_PROPERTY)]
class Slug extends Constraint
{
    public string $message = 'app.validator.slug.invalid';
}
```

And define the actual text in the YAML file:

```yaml
app.validator.slug.invalid: 'The value "{{ value }}" is not a valid slug.'
```

---

## 14. Advanced Scenarios {#advanced-scenarios}

### Scenario: Conditional Validation Based on Another Field

You want the `companyName` field to be required only when `accountType` is `'company'`.

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Context\ExecutionContextInterface;

class RegistrationRequest
{
    #[Assert\Choice(choices: ['individual', 'company'])]
    public string $accountType = 'individual';

    public ?string $companyName = null;

    #[Assert\Callback]
    public function validateCompanyName(ExecutionContextInterface $context): void
    {
        if ($this->accountType === 'company' && empty($this->companyName)) {
            $context->buildViolation('Company name is required for company accounts.')
                ->atPath('companyName')
                ->addViolation();
        }
    }
}
```

Alternatively, use the `#[Assert\When]` constraint introduced in Symfony 6.2:

```php
#[Assert\When(
    expression: 'this.accountType == "company"',
    constraints: [new Assert\NotBlank(message: 'Company name is required.')]
)]
public ?string $companyName = null;
```

---

### Scenario: Validating a Nested Collection of DTOs

You receive a JSON payload with an array of items and want to validate each one.

```php
class BatchCreateRequest
{
    /**
     * @var CreateProductRequest[]
     */
    #[Assert\Valid]
    #[Assert\Count(min: 1, max: 100)]
    #[Assert\All([new Assert\Type(CreateProductRequest::class)])]
    public array $products = [];
}
```

When the Serializer deserializes the array with proper type hints (using `ArrayDenormalizer` and typed arrays), the Validator's `#[Assert\Valid]` will cascade into each `CreateProductRequest` object.

---

### Scenario: Password Strength Validation

```php
use Symfony\Component\Validator\Constraints as Assert;

class ChangePasswordRequest
{
    #[Assert\Sequentially([
        new Assert\NotBlank(),
        new Assert\Length(min: 8, minMessage: 'Password must be at least 8 characters.'),
        new Assert\Regex(
            pattern: '/[A-Z]/',
            message: 'Password must contain at least one uppercase letter.'
        ),
        new Assert\Regex(
            pattern: '/[0-9]/',
            message: 'Password must contain at least one digit.'
        ),
        new Assert\Regex(
            pattern: '/[^a-zA-Z0-9]/',
            message: 'Password must contain at least one special character.'
        ),
    ])]
    public string $newPassword = '';

    #[Assert\EqualTo(
        propertyPath: 'newPassword',
        message: 'The confirmation password does not match.'
    )]
    public string $confirmPassword = '';
}
```

---

### Scenario: File Upload Validation

```php
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\Component\Validator\Constraints as Assert;

class ProfileUpdateRequest
{
    #[Assert\NotBlank]
    public string $displayName = '';

    #[Assert\Image(
        maxSize: '2M',
        maxSizeMessage: 'The avatar must be smaller than 2 MB.',
        mimeTypes: ['image/jpeg', 'image/png', 'image/webp'],
        mimeTypesMessage: 'Please upload a JPEG, PNG, or WebP image.',
        maxWidth: 2000,
        maxHeight: 2000,
        minWidth: 50,
        minHeight: 50
    )]
    public ?UploadedFile $avatar = null;
}
```

---

### Scenario: Cross-Field Date Validation

```php
use Symfony\Component\Validator\Constraints as Assert;

class ReservationRequest
{
    #[Assert\NotBlank]
    #[Assert\GreaterThanOrEqual('today')]
    public \DateTimeInterface $checkIn;

    #[Assert\NotBlank]
    #[Assert\GreaterThan(propertyPath: 'checkIn', message: 'Check-out must be after check-in.')]
    public \DateTimeInterface $checkOut;
}
```

---

### Scenario: Validating Polymorphic Payloads

When you have an API that accepts different payload shapes depending on a discriminator field, you can validate in the constructor or use a custom constraint:

```php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;

class ValidNotificationPayloadValidator extends ConstraintValidator
{
    public function validate(mixed $value, Constraint $constraint): void
    {
        if ($value->type === 'email' && empty($value->emailAddress)) {
            $this->context->buildViolation('Email address is required for email notifications.')
                ->atPath('emailAddress')
                ->addViolation();
        }

        if ($value->type === 'sms' && empty($value->phoneNumber)) {
            $this->context->buildViolation('Phone number is required for SMS notifications.')
                ->atPath('phoneNumber')
                ->addViolation();
        }
    }
}
```

---

### Scenario: Reusing Constraints via a Compound Constraint

When you apply the same set of constraints in multiple places, create a **compound constraint** to avoid duplication:

```php
namespace App\Validator;

use Symfony\Component\Validator\Constraints\Compound;
use Symfony\Component\Validator\Constraints as Assert;

#[\Attribute(\Attribute::TARGET_PROPERTY | \Attribute::TARGET_METHOD)]
class StrongPassword extends Compound
{
    protected function getConstraints(array $options): array
    {
        return [
            new Assert\NotBlank(),
            new Assert\Length(min: 8),
            new Assert\Regex('/[A-Z]/'),
            new Assert\Regex('/[0-9]/'),
            new Assert\Regex('/[^a-zA-Z0-9]/'),
        ];
    }
}
```

Usage:

```php
use App\Validator\StrongPassword;

class User
{
    #[StrongPassword]
    public string $password = '';
}
```

This is cleaner than repeating the same five constraints everywhere you have a password field.

---

### Scenario: Validating Query Parameters

```php
namespace App\Request;

use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\HttpKernel\Attribute\MapQueryString;

class SearchQuery
{
    #[Assert\Length(max: 255)]
    public string $q = '';

    #[Assert\Range(min: 1, max: 100)]
    public int $perPage = 20;

    #[Assert\Choice(choices: ['asc', 'desc'])]
    public string $order = 'asc';
}

// In a controller:
public function search(#[MapQueryString] SearchQuery $query): JsonResponse
{
    // $query is validated automatically
}
```

`#[MapQueryString]` was introduced in Symfony 6.3 and works similarly to `#[MapRequestPayload]`, but maps query string parameters.

---

## 15. Testing Validation {#testing-validation}

### Unit Testing a Custom Constraint

Symfony provides a `ConstraintValidatorTestCase` base class to simplify testing validators:

```php
namespace App\Tests\Validator;

use App\Validator\Slug;
use App\Validator\SlugValidator;
use Symfony\Component\Validator\Test\ConstraintValidatorTestCase;

class SlugValidatorTest extends ConstraintValidatorTestCase
{
    protected function createValidator(): SlugValidator
    {
        return new SlugValidator();
    }

    public function testValidSlug(): void
    {
        $this->validator->validate('my-valid-slug', new Slug());
        $this->assertNoViolation();
    }

    public function testInvalidSlug(): void
    {
        $this->validator->validate('Invalid Slug!', new Slug());
        $this->buildViolation('The value "{{ value }}" is not a valid slug.')
            ->setParameter('{{ value }}', 'Invalid Slug!')
            ->assertRaised();
    }

    public function testNullIsValid(): void
    {
        $this->validator->validate(null, new Slug());
        $this->assertNoViolation();
    }
}
```

### Integration Testing Validation in a Controller

Use `WebTestCase` to test your full controller flow including validation:

```php
namespace App\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class ProductControllerTest extends WebTestCase
{
    public function testCreateWithInvalidData(): void
    {
        $client = static::createClient();

        $client->request('POST', '/api/products', [], [], [
            'CONTENT_TYPE' => 'application/json',
        ], json_encode(['name' => '', 'price' => -10]));

        $this->assertResponseStatusCodeSame(422);

        $data = json_decode($client->getResponse()->getContent(), true);
        $this->assertArrayHasKey('errors', $data);
    }
}
```

### Testing Validation Directly on an Object

You can test validation without the HTTP layer by fetching the validator from the container:

```php
namespace App\Tests;

use App\Entity\User;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class UserValidationTest extends KernelTestCase
{
    public function testEmailMustBeValid(): void
    {
        self::bootKernel();
        $validator = static::getContainer()->get('validator');

        $user = new User();
        $user->name  = 'Alice';
        $user->email = 'not-an-email';

        $violations = $validator->validate($user);

        $this->assertCount(1, $violations);
        $this->assertSame('email', $violations[0]->getPropertyPath());
    }
}
```

---

## Quick Reference: Common Constraint Attributes

| Constraint     | Purpose             | Key Options                              |
| -------------- | ------------------- | ---------------------------------------- |
| `NotBlank`     | Not empty/null      | `message`                                |
| `NotNull`      | Not null            | `message`                                |
| `Length`       | String length       | `min`, `max`, `minMessage`, `maxMessage` |
| `Email`        | Valid email         | `mode`                                   |
| `Url`          | Valid URL           | `protocols`                              |
| `Regex`        | Pattern match       | `pattern`, `message`                     |
| `Range`        | Number/date range   | `min`, `max`                             |
| `Positive`     | Greater than 0      | `message`                                |
| `Choice`       | Allowed values      | `choices`, `multiple`                    |
| `Count`        | Array item count    | `min`, `max`                             |
| `All`          | Apply to each item  | `constraints`                            |
| `Valid`        | Cascade to nested   | -                                        |
| `Callback`     | Custom logic        | method name or callable                  |
| `Expression`   | Expression language | `expression`                             |
| `Sequentially` | Stop on first fail  | `constraints`                            |
| `AtLeastOneOf` | Pass at least one   | `constraints`                            |
| `When`         | Conditional         | `expression`, `constraints`              |
| `File`         | File upload         | `maxSize`, `mimeTypes`                   |
| `Image`        | Image upload        | `maxSize`, `maxWidth`, `maxHeight`       |

---

## Summary

Symfony 7.4's validation system is powerful and flexible. The most important concepts to keep in mind are:

**Use attributes** for defining constraints directly on your classes and properties. This is the most readable approach and keeps constraints co-located with the data they describe.

**Use DTOs** for API and form data rather than validating entities directly. This separates your domain model from external input concerns.

**Use Sequentially** to avoid redundant errors on a single field. If a value is blank, there is no reason to also report that it is too short.

**Use GroupSequence** to avoid cross-group errors. If basic checks fail, skip the advanced ones.

**Use Valid** whenever you have nested objects that need their own validation. Without it, nested objects are silently ignored by the validator.

**Create Compound constraints** for reusable combinations of rules, such as password strength requirements applied in multiple places.

**Inject services** into your custom validators to perform database lookups or other external checks without coupling the constraint itself to any infrastructure.
