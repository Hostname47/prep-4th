# Symfony Forms - TP3

Please note that a form FormType, could have no children, just a form with empty form controls, and that is the case for example in a form with only one submit button:

```php
class AddToWishlistType extends AbstractType
{
    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            // csrf is enabled by default @see config/packages/csrf.yaml:4
            'csrf_protection' => true, // Ensure to protect against the csrf attack
            'csrf_token_id' => 'add_to_wishlist',
            'csrf_field_name' => '__token',
        ]);
    }
}
```

in twig template:

```twig
<div class="card mb-4">
    <div class="card-body">
        <h3 class="card-title">{{ course.name | upper }}</h3>
        <h6 class="card-subtitle mb-2 text-muted">
            {{ course.category.name }} — par {{ course.author.name }}
        </h6>
        ....
        ....

	◘◘◘ {# Here is the content of the form (just a button) #} ◘◘◘
        {{ form_start(form) }}
	         <button type="submit" class="btn btn-primary">
	            Add to wish list
	         </button>
        {{ form_end(form) }}
    </div>
</div>
```

**Important**:
Sometimes, there are some forms that are appeared in more than one page, and we need a way to make those forms portable and easier to call in different places, and that by avoiding repeating ourselves. To do so, let's follow along:

1. Create the form type of the form (Email subscription form as an example)

```php
class SubscribeType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder->add(child: 'email', type: EmailType::class);
    }

    public function configureOptions(OptionsResolver $resolver): void
	{
	    $resolver->setDefaults(defaults: [
	        'csrf_token_id' => '__subscribe_form',
	        'csrf_field_name' => '_token_'
	    ]);
	}
}
```

2. Create twig template for the form:

```twig
{{ form_start(form, {action: url('app_subscribe') }) }}
    <h5>Subscribe to our newsletter</h5>
    <p>Monthly digest of what's new and exciting from us.</p>
    <div class="d-flex flex-column flex-sm-row w-100 gap-2">
        {{ form_label(form.email, 'Email address', {label_attr: {class: 'visually-hidden'}}) }}
        {{ form_widget(
            form.email,
            {attr: {class: 'form-control', placeholder: 'Email address'}}
        ) }}
        <button class="btn btn-primary" type="button">Subscribe</button>
        {{ form_errors(form) }}
    </div>
{{ form_end(form) }}
```

3.  Create **controller action** to render this form template: Since the form will appear across the entire website, it’s not practical to build and render it separately in every route and controller.  
    Instead, we create a **dedicated controller** responsible for generating and rendering the form, then we call that controller directly inside Twig with: `render(controller(...))`
    This allows the form to be reused anywhere in the templates without duplicating logic:
    ```php
    class SubscribeController extends AbstractController
    {
    public function showForm(): Response
    {
    $form = $this->createForm(SubscribeType::class);

                return $this->render(
                    'subscribe/index.html.twig',
                    ['form' => $form]
                );
            }
        }
        ```
        Please note that the action or the method responsible for generating and rendering the form, does not have a Route associated to it, and that is because its rtole is not render this form for a specific route, but used by pages that want this form and generate it to them using `render(controller(..))` method (see the next step)

4.  Call render(controller(...)) to render the form:

```php
<div class="col-md-5 offset-md-1 mb-3">
  {{ render(controller('App\\Controller\\SubscribeController::showForm')) }}
</div>
```

Now in each page whenever we need to render the subscription form, we just call the above snippet instead of repeating the twig template or create dedicated routes for it.

By default, **Symfony best practice** is to submit a form to the same controller action that rendered it. However, you can change the form’s action if needed.
