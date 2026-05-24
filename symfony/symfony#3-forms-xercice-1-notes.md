# Symfony Forms - TP1

Sometimes, forms contain many fields, so it is clean to create sub forms (FormTypes inside FormType) to make those portable and reusabole and to handle each form logic separatly.
Let's take the following example:

```php
class CreditCardType extends AbstractType
{
    public function
	    buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder->add(child: 'cardNumber', type: TextType::class)
            ->add(child: 'expirationDate', type: ExpirationDateType::class)
            ->add(child: 'cvv', type: TextType::class, options: [
                'attr' => [
                    'maxlength' => 3,
                ],
            ])
            ->add(child: 'billingAddress', type: BillingAddressType::class)
            ->add(child: 'shippingAddress', type: ShippingAddressType::class);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
    }
}
```

Here the main formtype, has 3 sub formtypes which are `ExpirationDateType`, `BillingAddressType` and `ShippingAddressType`.

Please note that **symfony support a special type**: `CountryType::class` which display all the world countries in a dropdown select element :

```php
public function buildForm(FormBuilderInterface $builder, array $options): void
{
  $builde
    ->add(child: 'country', type: CountryType::class)
    ->add(child: 'addressLine1', type: TextType::class, options: [
        'required' => false,
        'help' => 'Enter the number (ex: PO Box 123 would be: 123 PO Box)',
    ])
    ->add(child: 'addressLine2', type: TextType::class, options: [
        'required' => false,
    ])
    ->add(child: 'city', type: TextType::class)
    ->add(child: 'state', type: TextType::class)
    ->add(child: 'emailAddress', type: EmailType::class);
}
```

**-> To add a select (dropdown) formtype element to your form**, use ChoiceType as following:

```php
$builder
  // ... other fields
  ->add('priority', ChoiceType::class, [
      'choices' => [
          'High' => 'high',
          'Medium' => 'medium',
          'Low' => 'low',
      ],
      'placeholder' => 'Choose a priority',
      'label' => 'Task Priority',
  ]);
```

**-> To add a radio/checkbox buttons to your form**, use expended options within ChoiceType:

`expanded` is an option primarily used in ChoiceType (radio buttons/checkboxes) or EntityType to change the rendering from a default select dropdown (`<select>`) to a list of clickable, expanded HTML elements (`<input type="radio">` or `<input type="checkbox">`), making all choices visible at once.

**Key Aspects of the `expanded` Option:**

- **Behavior:** When `expanded` is `true`, it expands the choices.
- **Radio Buttons:** Used with `multiple => false` (default) to render radio buttons.
- **Checkboxes:** Used with `multiple => true` to render checkboxes.
- **Alternative:** When `expanded` is `false` (default), it renders a dropdown (`<select>`).
-
- **Usage Example:**

  ```php
  $builder->add('size', ChoiceType::class, [
      'choices' => ['Small' => 's', 'Large' => 'l'],
      'expanded' => true, // Renders as radio buttons
      'multiple' => false,
  ]);
  ```

  ```php
  $builder->add('size', ChoiceType::class, [
      'choices' => ['Small' => 's', 'Large' => 'l'],
      'expanded' => true, // Renders as checkbox buttons
      'multiple' => true,
  ]);
  ```

  ```php
  $builder->add('size', ChoiceType::class, [
      'choices' => ['Small' => 's', 'Large' => 'l'],
      'expanded' => false, // Default -> select (dropdown)
      'multiple' => true,
  ]);
  ```

Now let's create a controller to render our form:

```php
 #[Route(name: 'app_payment', path: '/pay')]
 public function index(): Response
 {
     $form = $this->createForm(CreditCardType::class);

     return $this->render('payment/index.html.twig', [
         'form' => $form,
     ]);
 }
```

Then we render it in our twig template:

```twig
 {% extends 'base.html.twig' %}
 {% block body %}
 {{ form(form) }}
 {% endblock body %}
```

But this will render it in a static way,m we need to take control to how this form will be look like, so we will m=not render it using `{{ form(form) }}`, but instead we will use helpers like form_start, form_row,... to customize the way it will displayed

Now we will begin by replacing our template with this:

```twig
 {% extends 'base.html.twig' %}
 {% form_theme form with [
    'form/themes/payment/billing_address/custom_billing_address_theme.html.twig',
    'form/themes/payment/custom_theme.html.twig',
    'form/themes/payment/shipping_address/custom_shipping_address.html.twig'
 ] %}
 {% block body %}
    {{ form_start(form) }}
    {{ form_row(form) }}
    <button type="submit" class="btn btn-primary">
        Pay now
    </button>
    {{ form_end(form) }}
 {% endblock body %}
```

**Please remember** that we can use **multiple themes** using **with [...]**

Now we will start customizing the parent form, and each subform using themes that we already mentioned above with **with**:

templates/form/themes/payment/billing_address/**custom_billing_address_theme.html.twig** :

```twig
{% block billing_address_row %}
    <div class="pt-5">
        {{ form_label(form, 'Billing Address', {label_attr: {class: 'offset-md-2 fw-bolder'}}) }}
        <div class="form-group row mb-2">
            {{ form_label(form.firstName, 'First Name: ' ,{label_attr: {class: 'col-2 col-form-label'}}) }}
            <div class="col-10">
                {{ form_widget(form.firstName) }}
            </div>
            {{ form_errors(form.firstName) }}
        </div>
        <div class="form-group row mb-2">
            {{ form_label(form.lastName, 'Last Name: ' ,{label_attr: {class: 'col-2 col-form-label'}}) }}
            <div class="col-10">
                {{ form_widget(form.lastName) }}
            </div>
            {{ form_errors(form.lastName) }}
        </div>
        <div class="form-group row mb-2">
            <div class="col-2">
                {{ form_label(form.country, 'Country: ' ,{label_attr: {class: 'col-2 col-form-label'}}) }}
            </div>
            <div class="col-10">
                {{ form_widget(form.country) }}
            </div>
            {{ form_errors(form.country) }}
        </div>
        <div class="form-group row mb-2">
            <div class="col-2">
                {{ form_label(form.addressLine1, 'Billing address: ' ,{label_attr: {class: 'col-2 col-form-label'}}) }}
            </div>
            <div class="col-10">
                {{ form_widget(form.addressLine1) }}
                {{ form_help(form.addressLine1) }}
            </div>

            {{ form_errors(form.addressLine1) }}
        </div>
        <div class="form-group row mb-2">
            <div class="col-2">
                {{ form_label(form.addressLine2) }}
            </div>
            <div class="col-10">
                {{ form_widget(form.addressLine2) }}
            </div>
            {{ form_errors(form.addressLine2) }}
        </div>
        <div class="form-group row mb-2">
            <div class="col-2">
                {{ form_label(form.city, 'City: ' ,{label_attr: {class: 'col-2 col-form-label'}}) }}
            </div>
            <div class="col-10">
                {{ form_widget(form.city) }}
            </div>
            {{ form_errors(form.city) }}
        </div>
        <div class="form-group row mb-2">
            <div class="col-2">
                {{ form_label(form.state, 'State: ' ,{label_attr: {class: 'col-2 col-form-label'}}) }}
            </div>
            <div class="col-10">
                {{ form_widget(form.state) }}
            </div>
            {{ form_errors(form.state) }}
        </div>
        <div class="form-group row mb-2">
            <div class="col-2">
                {{ form_label(form.zipCode, 'ZIP: ' ,{label_attr: {class: 'col-2 col-form-label'}}) }}
            </div>
            <div class="col-10">
                {{ form_widget(form.zipCode) }}
            </div>
            {{ form_errors(form.zipCode) }}
        </div>
        <div class="form-group row mb-2">
            <div class="col-2">
                {{ form_label(form.phoneNumber, 'Phone Number: ' ,{label_attr: {class: 'col-2 col-form-label'}}) }}
            </div>
            <div class="col-10">
                {{ form_widget(form.phoneNumber) }}
            </div>
            {{ form_errors(form.phoneNumber) }}
        </div>
        <div class="form-group row mb-2">
            <div class="col-2">
                {{ form_label(form.emailAddress, 'Email Address: ' ,{label_attr: {class: 'col-2 col-form-label'}}) }}
            </div>
            <div class="col-10">
                {{ form_widget(form.emailAddress) }}
            </div>
            {{ form_errors(form.emailAddress) }}
        </div>
    </div>
{% endblock billing_address_row %}

{% block _billing_address_label %}
    <label {{ block('label_attributes') }}>{{ label }}</label>
{% endblock  _billing_address_label %}
```

templates/form/themes/payment/shipping_address/**custom_shipping_address.html.twig**

```twig
{% block shipping_address_row %}
    <div class="pt-5">
        {{ form_label(form, 'Shipping Address', {label_attr: {class: 'offset-md-2 fw-bolder'}}) }}
    </div>
    <div class="form-group row mb-2">
        {{ form_label(form.shippingAddress, ' ' ,{label_attr: {class: 'col-2 col-form-label'}}) }}
        <div class="col-10">
            {{ form_widget(form.shippingAddress) }}
        </div>
        {{ form_errors(form.shippingAddress) }}
    </div>
{% endblock shipping_address_row %}
```

templates/form/themes/payment/**custom_theme.html.twig**

```twig
{% block _credit_card_expirationDate_row %}
    <div class="row">
        <div class="col-2">
            {{ form_label(form) }}
        </div>
        <div class="col-10">
            <div class="row">
                <div class="col-6">
                    {{ form_row(form.year) }}
                </div>
                <div class="col-6">
                    {{ form_row(form.month) }}
                </div>
            </div>
        </div>
    </div>
    {{ form_errors(form) }}
{% endblock  _credit_card_expirationDate_row %}

{% block _credit_card_cvv_row %}
    <div class="row">
        <div class="col-2">
            {{ form_label(form) }}
        </div>
        <div class="col-10">
            <div class="form-group">
                {{ form_widget(form) }}
            </div>
        </div>
        {{ form_errors(form) }}
    </div>
{% endblock _credit_card_cvv_row %}

{% block _credit_card_cardNumber_row %}
    <div class="row">
        <div class="col-2">
            {{ form_label(form) }}
        </div>
        <div class="col-10">
            <div class="form-group">
                {{ form_widget(form) }}
                <img src="{{ asset('images/img.png') }}" width="200px"/>
            </div>
        </div>
        {{ form_errors(form) }}
    </div>
{% endblock _credit_card_cardNumber_row %}
```

Now let's create FormType classes:

1. **BillingAddressType.php**

```php
<?php

declare(strict_types=1);

namespace App\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\CountryType;
use Symfony\Component\Form\Extension\Core\Type\EmailType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class BillingAddressType extends AbstractType
{
  public function buildForm(FormBuilderInterface $builder, array $options): void
  {
    $builder
        ->add(child: 'firstName', type: TextType::class)
        ->add(child: 'lastName', type: TextType::class)
        ->add(child: 'country', type: CountryType::class)
        ->add(child: 'addressLine1', type: TextType::class, options: [
            'required' => false,
            'help' => 'If your billing address is a PO Box, please enter the number first.',
        ])
        ->add(child: 'addressLine2', type: TextType::class, options: [
            'required' => false,
            'label' => false,
        ])
        ->add(child: 'city', type: TextType::class)
        ->add(child: 'state', type: TextType::class)
        ->add(child: 'zipCode', type: TextType::class)
        ->add(child: 'phoneNumber', type: TextType::class)
        ->add(child: 'emailAddress', type: EmailType::class);
  }

  public function configureOptions(OptionsResolver $resolver): void
  {
  }
}
```

2. **CreditCardType.php**

```php
<?php

declare(strict_types=1);

namespace App\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class CreditCardType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add(child: 'cardNumber', type: TextType::class, options: [
                'label' => 'Card Number :',
            ])
            ->add(child: 'expirationDate', type: ExpirationDateType::class, options: [
                'label' => 'Expiration Date :',
            ])
            ->add(child: 'cvv', type: TextType::class, options: [
                'attr' => [
                    'maxlength' => 3,
                ],
            ])
            ->add(child: 'billingAddress', type: BillingAddressType::class)
            ->add(child: 'shippingAddress', type: ShippingAddressType::class);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults(defaults: [
            'label' => false,
            'enable_csrf' => true,
            'csrf_token_id' => '__pay',
            'csrf_token_parameter' => '_pay_token',
        ]);
    }
}

```

3. **ExpirationDateType.php**

```php
<?php

declare(strict_types=1);

namespace App\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class ExpirationDateType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add(child: 'year', type: TextType::class, options: [
                'label' => 'yy',
            ])
            ->add(child: 'month', type: TextType::class, options: [
                'label' => 'mm',
            ]);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
    }
}
```

4. **ShippingAddressType.php**

```php
<?php

declare(strict_types=1);

namespace App\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\ChoiceType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class ShippingAddressType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add(child: 'shippingAddress', type: ChoiceType::class, options: [
                'choices' => [
                    'Same as billing address' => 'keep_same_address',
                    'Enter a different address' => 'enter_different_address',
                ],
                'expanded' => true,
                'multiple' => false,
            ]);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
    }
}
```

5. **PaymentController.php**

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Form\Type\CreditCardType;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class PaymentController extends AbstractController
{
    #[Route(name: 'app_payment', path: '/pay')]
    public function index(): Response
    {
        $form = $this->createForm(CreditCardType::class);

        return $this->render('payment/index.html.twig', [
            'form' => $form,
        ]);
    }
}
```
