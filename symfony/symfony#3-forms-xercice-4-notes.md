## Twig, Forms, Customization and Validation Notes on TPS

`Symfony\Component\Form\Extension\Core\Type\RepeatedType;` est utilisé pour définir un champ qui nécessite une confirmation.
l'option `novalidate` pour désactiver la validation `html5` coté front.
To add password with confirmation in Form types, use `RepeatedType` like the following:

```php
public function buildForm(FormBuilderInterface $builder, array $options): void
{
  $builder->add(child: 'email', type: EmailType::class)
    ->add(child: 'fullName', type: TextType::class)
    ->add(child: 'password', type: ◘◘=> RepeatedType::class, options: [
        'type' => PasswordType::class,
        ◘◘=>'first_options' => [
            'help_html' => true,
            'help' =>
                '<ul>
				 <li>Doit inclure au moins un chiffre</li>
				 <li>Must contains special chars like: `@`, `-`, `_`</li>
				 ...
			</ul>'
            ]
  ]);
}
```

The following code display the form and customize it (personalization)

```twig
{% extends 'base.html.twig' %}
# This is how we can use a theme that resides in the same file as the form
# and customize it [Note how we use _self to point to the same theme location]
{% form_theme registerForm _self %}
	# Note that registerForm above, is the name of parameter we pased
	# from controller to the twig template ($registerForm = $this->createForm)

{% block body %}
    {{ form_start(registerForm) }}
    <div class="mb-3">
        {{ form_row(registerForm.email) }}
    </div>
    <div class="mb-3">
        {{ form_row(registerForm.password) }}
    </div>
    <div class="mb-3">
        {{ form_row(registerForm.fullName) }}
    </div>
    <button type="submit" class="btn btn-primary">Register</button>
    {{ form_end(registerForm) }}
{% endblock body %}

{% block _register_password_row %}
    <div class="row">
        <div class="col-6">
            {{ form_row(form.first, {label: 'Password: '}) }}
        </div>
        <div class="col-6">
            {{ form_row(form.second, {label: 'Confirm Password: '}) }}
        </div>
    </div>
{% endblock _register_password_row %}

{% block _register_email_row %}
    <div class="row">
        <div class="col-12">
            {{ form_label(form) }}
            <div class="input-group mb-3">
                {{ form_widget(form) }}
                <span class="input-group-text" id="basic-addon2">@ehei.ac.ma</span>
            </div>
            {{ form_errors(form) }}
        </div>
    </div>
{% endblock _register_email_row %}
```

For clean code purposes, whenever you have a payload that Symfony receives, it is preferable to create a DTO for it to make code more readable and maintanable

Also, **validatiion of the payload, also can be done inside the DTO**. This way you gain the clarity and separate concerns to response the Single-Responsability principle

Usually, in a senior-level, and when dealing with 🔹 API / CQRS / Clean Architecture:
we usually use `Constructor + public readonly properties`

Example (very modern):

```php
final class RegistrationRequest
{
   public function __construct(
     public readonly string $email,
     public readonly string $password,
     public readonly string $fullName,
   ) {}
}
```

Now let's understand the flow from defining form structure to process it with DTOs:

1. Create a FormType

```php
class RegistrationType extends AbstractType
{
  public function buildForm(FormBuilderInterface $builder, array $options)
  {
	  $builder->add('email')->add('password')->add('fullName');
  }
}
```

2. Create DTO:

```php
class RegistrationRequest
{
	#[Assert\Email]
	#[Assert\NotBlank]
	private ?string $email;

	#[Assert\NotBlank]
	private ?string $password;

	#[Assert\NotBlank]
	private ?string $fullName;

	// getters / setters
}
```

3. Now let's process the form inside controller: (Really important)

```php
use App\DTO\RegistrationRequest;

public function register(Request $request, UserService $userService)
{
	$dto = new RegistrationRequest();

	$form = $this->createForm(RegistrationType::class, $dto);  ◘◘◘◘
	$form->handleRequest($request);   ◘◘◘◘

	if ($form->isSubmitted() && $form->isValid()) {

	$userService->register($dto);

	return $this->redirectToRoute('app_home');
	}

	return $this->render('register.html.twig', [
		'form' => $form->createView(),
	]);
}
```

---

To validate password, you can use Regex as following:

```php
#[Assert\Regex(
    message: 'Your password does not respect rules',
	pattern: '/^(?=.*[A-Z])(?=.*[a-z])(?=.*\d)(?=.*[@\-_]).{8,}$/')]
private ?string $password; # this property and validation can be in DTO
```

**Important note** (from TP):
In PHP, if you declare a property as `string` (non-nullable), PHP will throw a `TypeError` immediately if `null` is assigned to it. This happens before Symfony's validation system runs.

To allow Symfony's Validator (for example `#[Assert\NotNull]` or `#[Assert\NotBlank]`) to handle validation properly, the property must be declared as nullable (`?string`).

This way:

- PHP allows `null` during object hydration.
- Symfony Validator checks the value afterward.
- Validation errors are handled cleanly instead of causing a runtime crash.

In short:

Nullable types (`?string`) allow Symfony to control validation.  
Non-nullable types (`string`) make PHP enforce strict typing before validation occurs.

---

**To validate form**:

```php
#[Route('/register')]
public function index(Request $req): Response
{
    $signupDTO = new RegistrationRequest();

    $form = $this->createForm(type: RegisterType::class, data: $signupDTO);

    $form->handleRequest($req);

    if ($form->isSubmitted() && $form->isValid()) {
        // send registration email for confirmation
        dd(
            $form->getData(),
            $signupDTO,
            $form->getData() === $signupDTO
        );
    }

    return $this->render('register/index.html.twig', [
        'form' => $form,
    ]);
}
```

What does **handleRequest** do ?

It reads the HTTP request and, if the form was submitted, it:

1.  Extracts submitted form data from the request
2.  Maps that data into your bound object (`$registrationDTO`)
3.  Marks the form as submitted
4.  Triggers validation

```php
$register->handleRequest($req);
```

After this line:

- `$registrationDTO` is filled with submitted values (please note, that this happens because we passed the DTO as data argument when we called `createForm`)
- `$register->isSubmitted()` becomes `true` (if POST data exists)
- `$register->isValid()` runs Symfony validation
- `$register->getData()` returns the same `$registrationDTO` instance

That's why: `$register->getData() === $registrationDTO` returns `true`.

**Also Note**: Please note the importance of data_class when using DTOs along with forms

`data_class` tells Symfony **which PHP class the form should map its data to**.

**What it does ?** => When you set: `'data_class' => RegistrationRequest::class`,
Symfony will:

- Create (or use) an instance of `RegistrationRequest`
- Map form fields to its properties
- Call its setters
- Validate it as that object
- Return it via `$form->getData()`

**Without `data_class`: The form would return an **array\*\*, not an object.

---

You can combine two validation constraints into one, if you fell that the two are always used with each other, like in the case of NotNull and NotBlank, you can do that by creating a constraints class and extend **Compound** like following:

```php
<?php

namespace App\Validator\Constraints;

use Symfony\Component\Validator\Constraints\Compound;
use Symfony\Component\Validator\Constraints as Assert;

#[\Attribute]
class RequiredField extends Compound
{
    protected function getConstraints(array $options): array
    {
        return [
            new Assert\NotNull(message: 'Champ obligatoire'),
            new Assert\NotBlank(message: 'Ce champ ne doit pas etre vide'),
        ];
    }
}
```

The instruction `#[\Attribute]` tells PHP that this class can be used as an attribute.
It is the way attributes are defined (declared) in PHP.

Now you can use your constraints normally like any other constraint:

```php
#[Assert\Email(message: 'Cette adresse email est invalide.')]
#[RequiredField]
protected ?string $email;

#[RequiredField]
private ?string $fullName;
```

You can also **create your own constraints** like following:

1. First create a class that inherit from `Symfony\Component\Validator\Constraint` which will then represent you constraints class:

```php
use Symfony\Component\Validator\Constraint;

#[\Attribute]
class PasswordField extends Constraint
{
    public string $message = 'The password you provided is not compliant with our policies';

    public function __construct(
		?string $message = null,
		mixed $options = null,
		?array $groups = null,
		mixed $payload = null)
    {
        parent::__construct($options, $groups, $payload);

        $this->message = $message ?? $this->message;
    }
}
```

2. You need also to create the Validator for it:

The constraint class alone is not enough: you also need to create its validator. By default, the validator is a class with the same name as the constraint, suffixed with `Validator`. In our case, it is `PasswordFieldValidator`.
However, you can customize this behavior by overriding the `validatedBy` method in `PasswordField`.

```php
<?php

namespace App\Validator\Constraints;

use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;
use Symfony\Component\Validator\Exception\UnexpectedTypeException;
use Symfony\Component\Validator\Exception\UnexpectedValueException;

class PasswordFieldValidator extends  ConstraintValidator
{
    private const PASSWORD_PATTERN = '/^(?=.*[A-Z])(?=.*[a-z])...';

    public function validate(mixed $value, Constraint $constraint): void ◘◘
    {
        if (!$constraint instanceof PasswordField) {
            throw new UnexpectedTypeException($constraint, PasswordField::class);
        }

        // if the value of $value is null or empty, just do nothing. You must use NotNull and NotBlank.
        if (null === $value || '' === $value) {
            return;
        }

        // passwords can only be strings
        if (!\is_string($value)) {
            throw new UnexpectedValueException($value, 'string');
        }

        if (\preg_match(self::PASSWORD_PATTERN, $value) === 1) { // passowrd valid
            return;
        }

        // add a constraint violation to the list
        $this->context->buildViolation($constraint->message)
            ->addViolation();
    }
}
```

Now you can use as any other constraint attribute:

```php
use App\Validator\Constraints\PasswordField;

class RegistrationRequest
{

   //...

   #[PasswordField] ◘◘
   /**
    *  you can also specify message: #[PasswordField(
    *    message: "Password rules: The password must contain at least 8 chars,
    *    one uppercase, one loawercase, one special char..."
    *  )]
    */
   private ?string $password;

   // ...

}
```
