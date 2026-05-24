# Symfony Forms - TP2

1. First of all, lets create the DTO that the form will send:

   ```php
   <?php

   namespace App\DTO;

   use Symfony\Component\Validator\Constraints as Assert;

   class AddProductToCartDTO
   {
       public const COLORS = ['matte-black', 'pearl-white', 'silver'];

       #[Assert\NotBlank()]
       #[Assert\NotNull()]
       #[Assert\Positive()]
       #[Assert\LessThanOrEqual(10)]
       private ?int $quantity;

       #[Assert\NotBlank()]
       #[Assert\NotNull()]
       #[Assert\Choice(
           choices: self::COLORS,
           message: 'Choose a valid color.'
       )]
       private ?string $color;

       public function getQuantity(): int
       {
           return $this->quantity;
       }

       public function setQuantity(int $quantity): void
       {
           $this->quantity = $quantity;
       }

       public function getColor(): string
       {
           return $this->color;
       }

       public function setColor(string $color): void
       {
           $this->color = $color;
       }
   }
   ```

2. Then let's create the FormType:

   ```php
   <?php

   namespace App\Form\Type;

   use App\DTO\AddProductToCartDTO;
   use Symfony\Component\Form\AbstractType;
   use Symfony\Component\Form\Extension\Core\Type\ChoiceType;
   use Symfony\Component\Form\Extension\Core\Type\NumberType;
   use Symfony\Component\Form\FormBuilderInterface;
   use Symfony\Component\OptionsResolver\OptionsResolver;

   class ProductType extends AbstractType
   {
       public function buildForm(FormBuilderInterface $builder, array $options)
       {
           $builder->add(child: 'quantity', type: NumberType::class, options: [
                   'html5' => true,
                   'label' => 'Quantity',
               ])
               ->add(child: 'color', type: ChoiceType::class, options: [
                   'choices' => [
                       'Matte Black' => 'matte-black',
                       'Pearl White' => 'pearl-white',
                       'Silver'      => 'silver',
                   ],
                   'label' => 'Select a color',
               ]);
       }

       public function configureOptions(OptionsResolver $resolver)
       {
           $resolver->setDefaults([
               'data_class' => AddProductToCartDTO::class,
               'novalidate' => 'novalidate',
           ]);
       }
   }
   ```

3. Create the controller:

   ```php
   <?php

   namespace App\Controller;

   use App\DTO\AddProductToCartDTO;
   use App\Form\Type\ProductType;
   use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
   use Symfony\Component\HttpFoundation\Request;
   use Symfony\Component\HttpFoundation\Response;
   use Symfony\Component\Routing\Attribute\Route;

   final class ProductController extends AbstractController
   {
       #[Route('/product', name: 'app_product')]
       public function index(): Response
       {
           return $this->render('product/index.html.twig', []);
       }

       #[Route(path: '/cart', name: 'cart', methods: ['POST'])]
       public function cart(Request $request): Response
       {
           $dto = new AddProductToCartDTO();

           $form = $this->createForm(type: ProductType::class, data: $dto);

           $form->handleRequest($request);

           if ($form->isSubmitted() && $form->isValid()) {
               return $this->render('product/cart.html.twig', [
                   'data' => $dto,
               ]);
           }

           return $this->render('product/foo.html.twig', [
               'form' => $form,
           ]);
       }

       #[Route('/product/foo', name: 'foo_product')]
       public function foo(): Response
       {
           $dto = new AddProductToCartDTO();

           $form = $this->createForm(type: ProductType::class, data: $dto);

           return $this->render('product/foo.html.twig', [
               'form' => $form,
           ]);
       }
   }
   ```

4. Then we will create the twig responsible for rendering the formtype:

   ```twig
   {% extends 'base.html.twig' %}

   {% block title %}Product Details{% endblock %}

   {% block stylesheets %}
       {{ parent() }}
   {% endblock %}

   {% block body %}

   <div class="container my-5">
       <div class="row">
           <div class="col-md-6">
               <img
                   src="https://images.pexels.com/photos/90946/pexels-photo-90946.jpeg?auto=compress&cs=tinysrgb&w=800"
                   class="img-fluid rounded"
                   alt="Product"
               >
           </div>
           <div class="col-md-6">
               <h1 class="mb-3">Premium Wireless Headphones</h1>
               <p class="text-muted fs-4 mb-3">$129.99</p>
               <p class="mb-4">
                   Experience superior sound quality with our premium wireless headphones.
                   Features active noise cancellation, 30-hour battery life, and premium comfort padding.
               </p>
               <ul class="list-unstyled mb-4">
                   <li><strong>Brand:</strong> AudioTech</li>
                   <li><strong>Color:</strong> Matte Black</li>
                   <li><strong>Connectivity:</strong> Bluetooth 5.0</li>
                   <li><strong>Battery Life:</strong> 30 hours</li>
               </ul>

               {{ form_start(form, {action: url("cart")}) }}
                   {{ form_row(form.quantity) }}
                   {{ form_row(form.color) }}
                   <button type="submit" class="btn btn-primary">Add To Cart</button>
               {{ form_end(form) }}
           </div>
       </div>
   </div>

   {% endblock %}
   ```
