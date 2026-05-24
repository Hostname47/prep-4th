# Symfony Databases - Complete Guide [From classroom cours only]

Doctrine is the default ORM used in symfony; A php library that facilitate the process between symfony and database and also help with the mapping between database tables and php entities.

Doctrie consists with 2 major parts:

1. DBAL (Database Abstraction Layer): an advanced communication with relational databases, and it is compared with like PDO but more powerful
2. ORM that map tables within database into PHP classes (entities)
   Doctrine supports relational databases as well as NOSQL databases (MongoDB..)

### Database Configuration

You start by configuring DATABASE_URL environement file in .env

    DATABASE_URL="mysql://user:password@127.0.0.1:3306/my_database?serverVersion=8.0"

- `mysql` → database driver
- `user` → username
- `password` → password
- `127.0.0.1` → database host
- `3306` → port
- `my_database` → database name
- `serverVersion=8.0` → database server version

To configure Doctrine, you can do that in **config/packages/doctrine.yaml**
To get the options supported to configure, run:
`php bin/console debug:config doctrine` or `php bin/console config:dump-reference doctrine`

Please note that the DATABASE_URL is used in doctrine.yaml config file,

```yaml
	# config/packages/doctrine.yaml
	doctrine:
		dbal:
			url: "%env(resolve:DATABASE_URL)%"
			profiling_collect_backtrace: "%kernel.debug%"
			use_savepoints: true

	# You can also configure it using the following:

	doctrine:
		dbal:
			driver: 'pdo_mysql'
			host: '127.0.0.1'
			port: 3306
			dbname: 'my_database'
			user: 'my_user'
			password: 'my_password'
			server_version: '8.0'

		charset: utf8mb4
```

To create the database: **php bin/console doctrine:database:create**
To delete the database: **php bin/console doctrine:database:drop -f**
To see the list of doctrine commands: **php bin/console list doctrine**

An entity is a PHP class that reflect a table in database (look at entities in the previous guide if you jump here directly)

You have some doctrine annotations placed above class like #[ORM\Entity] to tell doctrine that this class is a reflected class on a table in database, and within the entity you define tabel columns with an annotation #[ORM\Column..]

+----------------------+
| PHP Class Entity |
|----------------------|
| class User |
| - id: int |
| - email: string |
+----------+-----------+
|
| (Mapping metadata:
| Attributes / Annotations / YAML)
v
+----------------------+
| Doctrine ORM |
|----------------------|
| Reads metadata |
| Builds mapping |
| Generates SQL |
+----------+-----------+
|
| (Schema synchronization)
v
+----------------------+
| Database Table |
|----------------------|
| user |
|----------------------|
| id INT (PK) |
| email VARCHAR(255) |
+----------------------+

To creeate an entity: **php bin/console nake:entity Book**
While creating the entity, Symfony will ask you about each column in the entity and if you are not sure about the type, you can type ? to list all types then choose.

**To update the entity** to add some other fields, you can **rerun** the previous command.

The **attributes #[ORM\*]** are the way how doctrine does the **mapping** between the class attributes and database table columns.

**#[ORM\Column]** is the way how you can map a class attr with a table column

You can even explicitly define the column name that will be used in database using **name** option

```php
/**
 * The database column will be named 'track_count'
 */
#[ORM\Column(name: 'track_count', type: 'integer')]
private ?int $trackCount = null;
```

**#[ORM\Id]** is the way to precise which attr to be used as PK. You can also create a composite PK as following:

```php
#[ORM\Entity]
class CarPart {
    #[ORM\Id]
    #[ORM\ManyToOne(targetEntity: Car::class)]
    private $car;

    #[ORM\Id]
    #[ORM\Column(type: "string")]
    private $partName;
}

```

**#[ORM\GeneratedValue]**: auto-increment field

**#[ORM\Entity]**: Tell doctrine that the class is an Entity

If your class name (Foo) is not the same as table name (boo) you can use: **#[ORM\Table(name: '...')]** as following:

```php

#[ORM\Entity(repositoryClass: ProductRepository::class)]
#[ORM\Table(name: "shop_products")] // Different from class name
class  Product
{
	#[ORM\Id]
	#[ORM\GeneratedValue]
	#[ORM\Column(type: "integer")]
	private  ?int  $id  =  null;

	public  function  getId(): ?int
	{
		return $this->id;
	}
}
```

### Migrations:

In **Symfony** with **Doctrine**, migrations are used to **manage and version database schema changes** in a safe and reproducible way, avoiding manual modifications or risky automatic updates.

Any structural change to an entity (adding, modifying, or removing a property) requires creating a migration so the change is actually applied to the database.

To create a migration: **php bin/console make:migration**

The generated migration contains the **delta (difference)** between the current database schema and the current state of your entities.

To apply the migration(s) to the database: **php bin/console doctrine:migrations:migrate**

In **Symfony** with **Doctrine**, writing to the database is done using the **EntityManager** service, which can be injected via autowiring by typing the argument as:

`Doctrine\ORM\EntityManagerInterface`

---

### Main Methods

- `$manager->persist($book)`  
  Tells Doctrine to insert a new row into the database. (please NOTE THAT persist does not save the entity into database, it just inform doctrine that we want to add it; it is the jub of flush who take care of insert the entity into database)
- For updates, you do **not** need to call `persist()` again.  
  Simply modify the `$book` object and call:
- `$manager->flush()` <= This is responsible to either create and/or update the database by taking what you persist or changed in your entities before call it.


    Doctrine automatically detects changes and decides whether to execute `INSERT INTO` or `UPDATE`.

- `$manager->remove($book)`

  Tells Doctrine to delete the corresponding row from the database.

### Example of save a book from form submission

Code

**Line 80** — Create the form by binding `BookType` to a new `Book` instance.
**Line 82** — Attach the form to the current HTTP request.
**Lines 84–91** — If submitted and valid, persist to the database and redirect to the book detail page.
**Lines 93–97** — Otherwise, render the Twig template and pass the form to the view.

```php
#[Route(path: '/books/add', methods: ['GET', 'POST'], name: 'app.books.add', priority: 1)]
public function addBook(Request $request, EntityManagerInterface $manager): Response
{
    $book = new Book();

    $form = $this->createForm(BookType::class, $book);        // Line 80

    $form->handleRequest($request);                           // Line 82

    if ($form->isSubmitted() && $form->isValid()) {           // Line 84
        $manager->persist($book);
        $manager->flush();

        return $this->redirectToRoute(
            route: 'app.books.view.isbn',
            parameters: ['isbn' => $book->getIsbn()],
        );
    }                                                         // Line 91

    return $this->render(                                     // Line 93
        view: 'book/add.html.twig',
        parameters: ['form' => $form],
    );                                                        // Line 97
}

```

### Explanation:

- **Line 80** — `createForm()` binds the `BookType` form class to the new `Book` entity instance.
- **Line 82** — `handleRequest()` attaches the form to the incoming HTTP request and populates the entity with submitted data.
- **Lines 84–91** — If the form is submitted and passes validation, the book is persisted to the database and the user is redirected to the detail page.
- **Lines 93–97** — If the form has not been submitted yet, the Twig template is rendered with the form passed as a parameter.

## Reading an Entry from the Database

In **Symfony** with **Doctrine**, to modify a record in the database, you must first retrieve it, then use it (for example, to display it in a form).

There are several ways to fetch objects from the database.

Data retrieval is generally done through a **Repository**, following the Repository Design Pattern.

A repository class is linked to its entity using: `#[ORM\Entity(repositoryClass: YourRepo::class)]`

Important rule:

> One entity = one repository class.  
> You cannot map a single entity to multiple repository classes.

### -> Repository Class

In **Doctrine** (inside a **Symfony** project), the **Repository class** — configured using:

    #[ORM\Entity(repositoryClass: YourRepository::class)]

— is responsible for retrieving records from the database and writing custom or complex queries.

### Built-in Retrieval Methods

Doctrine provides basic methods for quickly fetching data:

- **`find($id)`** : Returns one entity by its primary key.
  ```php
  $product = $productRepository->find(1);
  // SQL equivalent:
  // SELECT * FROM product WHERE id = 1;
  ```
- **`findAll()`** : Returns all records of the entity.
  ```php
  $products = $productRepository->findAll();
  // SQL equivalent:
  // SELECT * FROM product;
  ```
- **`findBy(array $criteria, array $orderBy = null, int $limit = null, int $offset = null)`** : Returns many results by conditions
  ```php
  $products = $productRepository->findBy(['isPublished' => true]);
  // SQL: SELECT * FROM product WHERE is_published = 1;
  $products = $productRepository->findBy(['isPublished' => true], ['price' => 'DESC']);
  // Above is published products ordered by price DESC
  $products = $productRepository->findBy(['isPublished' => true],['price' => 'DESC'], 5, 10);
  // Above: get all products that are published, order them by price desc, limit by 5 and offset 10
  ```
- **`findOneBy(criteria)`** : Returns one result by conditions
  ```php
  $product = $productRepository->findOneBy(['name' => 'iPhone 15']);
  // SQL: SELECT * FROM product WHERE name = 'iPhone 15' LIMIT 1;
  ```

That for reading from database, now let's jump into **edit**

---

### Edit an entry in Doctrine:

Before editing an entry you need first to fetch it, and after that, you can edit it either by calling setters, or by getting a form payload and hydrate the entity with the new data by calling handleRequest, then you can call $this->em->flush() to update the entry in database:

Example of taking form paload and update the entity:

```php
#[Route(
    path: '/books/{id}/edit',
    name: 'app_books_edit',
    methods: ['GET', 'POST'],
    requirements: ['id' => '\d+']
)]
public function edit(int $id, Request $request): Response
{
    $repository = $this->entityManager->getRepository(Book::class);
    $book = $repository->find($id);

    if (null === $book) { throw $this->createNotFoundException('No book found'); }

    $form = $this->createForm(BookType::class, $book);
    $form->handleRequest($request);

    if ($form->isSubmitted() && $form->isValid()) {
        $this->entityManager->flush();

        return $this->redirectToRoute('app_books_view_isbn', [
            'isbn' => $book->getIsbn(),
        ]);
    }

    return $this->render('book/edit.html.twig', [
        'form' => $form,
    ]);
}
```

Please note that in this example, we only update the entity if the user submit the form not in the controller by default (look at isSubmitted and isValid)

-> You can also just use setters to update the entity and flush.

Also **Remember**: It was possible to call `persist()` before `flush()` during an update as well, but it is not required because Doctrine already knows that this is a modification and not a new insertion (`persist()` in this case would have no effect).

## Delete an Entry:

To delete a record from the database, you simply call the `remove()` method of the `EntityManager` service, then confirm the changes by calling the `flush()` method.

The following code is just a basic deletion example. It is recommended to add CSRF validation for security.

```php
#[Route('/books/{id}/delete', name: 'app_books_delete', methods: ['POST'])]
public function delete(int $id): Response
{
	 $book = $this->entityManager  // fetch the entity from the database
				  ->getRepository(Book::class)
				  ->find($id);

	 if (!$book) {
		 throw $this->createNotFoundException('Book not found');
	 }

	 $this->entityManager->remove($book); // marks the entity for deletion
	 $this->entityManager->flush();       // executes the SQL `DELETE` query

	 return $this->redirectToRoute('app_books_list');
}
```

---

## Entity Mapping:

Usually, in the **_classical way_** of getting an entity to delete or edit, we use repository::find to get the entity then we check if the entity exists or not, if yes then work with it, otherwise we throw Not found exception:

```php
	$book = $this->entityManager
		->getRepository(Book::class)
		->find($id);

	if (!$book) {
		throw $this->createNotFoundException('Book not found');
	}

	// continue logic...
```

This works, but it is repetitive.

#### -> Let's use Automatic Entity Mapping (ParamConverter behavior in Symfony 7)

Symfony can automatically fetch the entity for you. But you should do the following:

- The `{id}` route parameter is required.
- The parameter name must match the entity property you want to use (here: `id`).
- The controller argument is typed as `Book`.

Because the argument is typed as `Book`, Symfony **automatically** queries the database.
If no object is found with the given `id`, a **404 error is automatically triggered**.

```php
#[Route('/books/{id}/edit', name: 'app_books_edit')]
public function edit(Book $book): Response
{
	// If no book is found, Symfony automatically throws 404

	// continue logic...
}
```

No `find()`.  
No manual 404 check.  
Much cleaner.

#### -> You can also use the following way:

```php
#[Route('/books/{isbn:book}', name: 'app_book_show')]
public function show(Book $book): Response
{
 return new Response($book->getTitle());
}
```

This way we can get the entity using a field other than id. (isbn in this example)

**What does `{isbn:book}` mean?** It means:

- `isbn` → route parameter
- `book` → entity argument name
- Symfony automatically uses: `findOneBy(['isbn' => value])`

So it maps:

    Route param: isbn
    Entity property: isbn
    Injected variable: $book

## More Control with #[MapEntity]

If you want more control over how the entity is retrieved, you can use `#[MapEntity]`.
This allows you to define how the route parameter maps to the entity.

### Example A — Mapping a Different Route Parameter

Suppose your route uses `{bookId}` instead of `{id}`.

### Without MapEntity (this would NOT work automatically)

```php
#[Route('/books/{bookId}/edit')]
public function edit(Book $book) {}
```

Symfony does not know that `bookId` should match `id`.

### With MapEntity

```php
use Symfony\Bridge\Doctrine\Attribute\MapEntity;

#[Route('/books/{bookId}/edit')]
public function edit(
	 #[MapEntity(mapping: ['bookId' => 'id'])] Book $book
): Response {
	 // Symfony now knows:
	 // route param "bookId" matches entity property "id"
}
```

---

### Example B — Find by Another Field (e.g., slug)

```php
#[Route('/books/{slug}', name: 'app_books_show')]
public function show(#[MapEntity(mapping: ['slug' => 'slug'])] Book $book): Response {
 // Finds Book where slug = {slug}
}
```

### Using a Custom Repository Method with `MapEntity`

Suppose we have a custom method to retrieve an object according to our own logic (for example: retrieve a book by its ID **or** by its ISBN).

This method must be defined inside the `BookRepository` class.

To reuse this function inside a controller argument, we configure the `expr` option of `#[MapEntity]`.

-> **Custom Repository Method**

```php
// src/Repository/BookRepository.php

public function getBookByStandardNumberOrId(string $value): ?Book
{
	 return $this->createQueryBuilder('b')
		 ->where('b.id = :value')
		 ->orWhere('b.isbn = :value')
		 ->setParameter('value', $value)
		 ->getQuery()
		 ->getOneOrNullResult();
}
```

This method searches by `id` OR by `isbn` and it returns one Book or null.
Now we can use that method in the controller to map the entity returned by it using expr (ExpressionLanguage provided by symfony) as following:

-- **Using `expr` in MapEntity**

```php
use Symfony\Bridge\Doctrine\Attribute\MapEntity;

#[Route('/books/{value}', name: 'app_books_show')]
public function show(
	 #[MapEntity(expr: 'repository.getBookByStandardNumberOrId(value)')]
	 Book $book
): Response {
	 // If nothing is found → automatic 404
	 return $this->render('book/show.html.twig', ['book' => $book]);
}
```

- `repository` refers to `BookRepository`
- `value` refers to the route parameter `{value}`
- Symfony calls: `BookRepository::getBookByStandardNumberOrId($value)`

If it returns `null`, Symfony automatically throws a 404.

This gives you full control over how the entity is retrieved.

### Injecting a Collection Instead of a Single Object

It is also possible to inject **a collection of entities**, not only a single one.

For example: inject all books published in 1997.

### Repository method (optional but clean)

```php
public function findBooksPublishedInYear(int $year): array
{
	 return $this->findBy(['publicationYear' => $year]);
}
```

```php
#[Route('/books/published/{year}', name: 'app_books_by_year')]
public function booksByYear(
	 #[MapEntity(expr: 'repository.findBy({"publicationYear": year})')]
	 array $books
): Response {
	 return $this->render('book/list.html.twig', [
		 'books' => $books,
	 ]);
}
```

- `year` comes from the route
- `findBy()` is called internally
- Symfony injects an array of Book objects

If no books are found:

- You simply get an empty array
- No 404 is triggered (because it’s a collection)

## Relationships between Entities

- **OneToOne**
- **ManyToOne – OneToMany**
- **ManyToMany**

### 1. OneToOne

A **OneToOne** relationship manages a 1–1 relation between two tables.
Example: An **Author** has one and only one Profile.

Suppose we already have an `Author` entity and we want to store additional information about the author's profile such as: Date of birth, Nationality and Biography we therefore need **two entities**: `Author` and `Profile`.

We can use the `#[ORM\OneToOne]` attribute to link one `Author` to one `Profile` or we can do `php bin/console make:entity` to update the Author entity and add the relationship using interactive shell console.

Reminder: after updating entities, generate and run migrations:

```bash
php bin/console make:migration
php bin/console doctrine:migrations:migrate
```

**Simple OneToOne Example**:

```php
#[ORM\Entity]
class Profile
{
 #[ORM\Id]
 #[ORM\GeneratedValue]
 #[ORM\Column]
 private ?int $id = null;

 #[ORM\Column(type: 'date', nullable: true)]
 private ?\DateTimeInterface $birthDate = null;

 #[ORM\Column(length: 100, nullable: true)]
 private ?string $nationality = null;

 #[ORM\Column(type: 'text', nullable: true)]
 private ?string $biography = null;

 #[ORM\OneToOne(inversedBy: 'profile')]
 #[ORM\JoinColumn(nullable: false)]
 private ?Author $author = null;

 // getters & setters ...
}
```

```php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Author
{
 #[ORM\Id]
 #[ORM\GeneratedValue]
 #[ORM\Column]
 private ?int $id = null;

 #[ORM\Column(length: 255)]
 private string $name;

 #[ORM\OneToOne(mappedBy: 'author', cascade: ['persist', 'remove'])]
 private ?Profile $profile = null;

 // getters & setters ...
}
```

**-> Important Concepts**

**- - > Owning Side vs Inverse Side**

In a OneToOne relation:

- The side with `#[ORM\JoinColumn]` is the **owning side** (Profile here)
- The other side is the **inverse side** (`mappedBy`).

The owning side is the one that contains the foreign key in the database.

**- - > About the “Yes” Question in make:entity**

When using:

php bin/console make:entity

Symfony asks:

> Do you want to add a new property to Author so that you can access Profile from it?

If you answer **Yes**:

- Symfony creates the reverse relation automatically.
- You can access:

```php
$profile->getAuthor();
$author->getProfile();
```

If you answer **No**:

- The relation is unidirectional.
- You can access only from one side.

It is not mandatory to have both sides, but it is often useful.

**Very Simple Usage Example:**

Creating an Author with a Profile:

```php
$author = new Author();
$author->setName('John Doe');

$profile = new Profile();
$profile->setNationality('French');
$profile->setAuthor($author);

$author->setProfile($profile);

$entityManager->persist($author);
$entityManager->flush();
```

Because of `cascade: ['persist']`, persisting the author also saves the profile.

👉 **Profile contains the foreign key**, not Author.  
👉 Therefore, **Profile is the owning side**.  
👉 Author is the inverse side (`mappedBy`).

**- - > The Rule You Must Remember :**

In Doctrine:

- The entity that has `#[ORM\JoinColumn]`  
  ➜ **is the owning side**  
  ➜ **contains the foreign key in the database**
- The entity that has `mappedBy`  
  ➜ is the inverse side  
  ➜ does NOT contain the foreign key

### 2. OneToMany / ManyToOne Relationships:

They represent relationships of type: (n, 0/1) or (n, 0/1). It is a single relationship viewed from two different sides. Example:

- One Author writes 0 to many Books (One Author → Many Books)
- One Book is written by one and only one Author (Many Books → One Author)

The entity that controls the relationship is called the **owning entity**. To configure these relations, we use: `#[ManyToOne]` and `#[OneToMany]`

**- - > Important note**:

- `ManyToOne` does NOT require `OneToMany`.
- But `OneToMany` requires `ManyToOne`.

### Understanding Both Sides

**-> Owning Side (ManyToOne)**

This side contains the foreign key in the database. In our example:

- `Book` is the owning side.
- It contains the `author_id` column.

**-> Inverse Side (OneToMany)**

This side reflects the relationship but does not store the foreign key. In our example:

- `Author` is the inverse side.
- It contains a collection of books.

### Example Implementation

**- -> Author Entity (Inverse Side)**

```php
#[ORM\Entity]
class Author
{
	 #[ORM\Id]
	 #[ORM\GeneratedValue]
	 #[ORM\Column]
	 private ?int $id = null;

	 #[ORM\Column(length: 255)]
	 private string $name;

	 #[ORM\OneToMany(mappedBy: 'author', targetEntity: Book::class)]
	 private Collection $books;

	 public function __construct()
	 {
		 $this->books = new ArrayCollection();
	 }

	 public function getBooks(): Collection
	 {
		 return $this->books;
	 }
}
```

**Explanation**

- `mappedBy: 'author'` This means: the `author` property inside the `Book` entity owns the relationship. So `Book::author` is the owning side.

---

**- - > Book Entity (Owning Side)**

```php
#[ORM\Entity]
class Book
{
	#[ORM\Id]
	#[ORM\GeneratedValue]
	#[ORM\Column]
	private ?int $id = null;

	#[ORM\Column(length: 255)]
	private string $title;

	#[ORM\ManyToOne(inversedBy: 'books')]
	#[ORM\JoinColumn(name: 'author_id', referencedColumnName: 'id', nullable: false)]
	private ?Author $author = null;

	public function getAuthor(): ?Author
	{
		return $this->author;
	}

	public function setAuthor(?Author $author): self
	{
		$this->author = $author;
		return $this;
	}
}
```

---

### Important Concepts

**-> mappedBy** Used on the inverse side (`OneToMany`). It tells Doctrine:

> Which property in the other entity owns the relationship?

In our case: `mappedBy: 'author'` because `Book` has: `private ?Author $author;`

**-> inversedBy** Used on the owning side (`ManyToOne`). It tells Doctrine:

> Which property in the other entity represents the inverse side?

In our case: inversedBy: 'books' because `Author` has: `private Collection $books;`

**-> JoinColumn** defines the foreign key column in the database. Example:

```php
#[ORM\JoinColumn(
 name: 'author_id',
 referencedColumnName: 'id',
 nullable: false
)]
```

### Important options:

- `name` The foreign key column name in the current table → `author_id`
- `referencedColumnName` The column in the target table (default = `id`)
- `nullable` Whether the relationship can be null
  - `false` → every book must have an author
  - `true` → a book may exist without an author

### Database Result

This configuration produces:
-> Author table: `id | name`
-> Book table:-- `id | title | author_id`

`author_id` is the foreign key referencing `author.id`.

---

**--<> Why ManyToOne Does Not Require OneToMany**

You can define only this:

```php
#[ORM\ManyToOne]
private ?Author $author = null;
```

And it works perfectly. But if you define only `OneToMany` without `ManyToOne`, Doctrine has no foreign key to manage. That’s why `OneToMany` requires a `ManyToOne`.

---

## EntityType

### Retrieving Choices from the Database

When working with `ManyToOne / OneToMany` relationships in Symfony forms, we use a special form type called: **EntityType**

It is a choice field type where the _options_ are automatically loaded from the database.

### Example: Add / Edit a Book with Author Selection

We have:

- `Book` → ManyToOne → `Author` (many books to one author)
- We want to select the author from the database when creating or editing a book.

**- - > BookType Form Example**

```php
// src/Form/BookType.php

use App\Entity\Book;
use App\Entity\Author;
use Symfony\Bridge\Doctrine\Form\Type\EntityType;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class BookType extends AbstractType
{
	public function buildForm(FormBuilderInterface $builder, array $options): void
	{
	 $builder
		 ->add('title')
		 ->add('author', EntityType::class, [
			 'class' => Author::class,
			 'choice_label' => 'name', // field shown in dropdown
			 'placeholder' => 'Choose an author'
		 ]);
	}

	public function configureOptions(OptionsResolver $resolver): void
	{
		$resolver->setDefaults([
		'data_class' => Book::class,
		]);
	}
}
```

---

**-> What Happens Here?**

- `'class' => Author::class` → fetches all authors from DB
- `'choice_label' => 'name'` → displays the author name
- Symfony automatically:
  - Loads authors
  - Displays them in a select dropdown
  - Converts the selected ID into an `Author` object

You don’t manually fetch authors.

**- - > lambda (closure) for choice_label**

Suppose: `Author` has a relation `profile`, and `Profile` has a method `getName()` and we want to display the **profile name** in the dropdown instead of `Author::name` :

```php
use App\Entity\Author;
use Symfony\Bridge\Doctrine\Form\Type\EntityType;

$builder->add('author', EntityType::class, [
    'class' => Author::class,
    'choice_label' => function (Author $author) {
        return $author->getProfile()->getName();
    },
]);
```

---

### Retrieving Nested Objects

When retrieving related objects: First, you retrieve the main object (`Book`), then call its getter to access the child object (`Author`). Example:

```php
$book = $bookRepository->find(1);
$author = $book->getAuthor();
```

**- - > Important Concept: Lazy Loading**

When you fetch the `Book`, Doctrine does NOT automatically fully load the related `Author`. Instead, it loads a: Proxy object **<--** This is called **Lazy Loading**.

It means:

- Doctrine delays the SQL query
- The related object is only loaded when you actually access it

Proxy here is: A lightweight placeholder object that looks like the real entity but is not fully loaded yet. It contains: The identifier (e.g., `author_id`), and logic that says: “If someone tries to use me, then load the real data from the database.”

**Example:**
$book = $bookRepository->find(1);

At this moment:

- The `Book` is fully loaded
- The `Author` is only a proxy (not fully loaded yet)

**When Is Author Actually Loaded?**

When you explicitly access it: `$book->getAuthor()->getName();`. Now Doctrine executes a new SQL query to fetch the Author. If Author has another relation like Profile:

    $book->getAuthor()->getProfile()->getName();

Each level may trigger another query if not already loaded.

Same above procedure applies for the Inverse Side !

**Example:**

`$author = $authorRepository->find(1);` At this moment:

- `Author` is loaded
- `books` collection is NOT loaded yet

Only when you do: `$author->getBooks();` Doctrine executes the SQL query to fetch books.

## ManyToMany Relationship

Used to represent relationships of type **(n, m)**. Example: A Book can belong to **one or many Genres** and a genre can contain **zero or many Books** This is a **ManyToMany** relationship.

### Example: Book ↔ Genre

**- - > Book Entity (Owning Side)**

```php
#[ORM\Entity]
class Book
{
	#[ORM\Id]
	#[ORM\GeneratedValue]
	#[ORM\Column]
	private ?int $id = null;

	#[ORM\Column(length: 255)]
	private string $title;

	#[ORM\ManyToMany(targetEntity: Genre::class, inversedBy: 'books')]
	#[ORM\JoinTable(name: 'book_genre')]
	private Collection $genres;

	public function __construct()
	{
		$this->genres = new ArrayCollection();
	}

	public function getGenres(): Collection
	{
		return $this->genres;
	}
}
```

**- - > Genre Entity (Inverse Side)**

```php
#[ORM\Entity]
class Genre
{
	#[ORM\Id]
	#[ORM\GeneratedValue]
	#[ORM\Column]
	private ?int $id = null;

	#[ORM\Column(length: 255)]
	private string $name;

	#[ORM\ManyToMany(mappedBy: 'genres', targetEntity: Book::class)]
	private Collection $books;

	public function __construct()
	{
		$this->books = new ArrayCollection();
	}
}
```

**- - > Database Result**

Doctrine creates a **join table** automatically:

**book_genre** (`book_id | genre_id`) **<-** This table connects both entities.

---

### Using ManyToMany in Forms (EntityType)

To manage ManyToMany in a form, we use: EntityType::class as shown previously

**- - > Example in BookType**

```php
use App\Entity\Genre;
use Symfony\Bridge\Doctrine\Form\Type\EntityType;

$builder->add('genres', EntityType::class, [
	'class' => Genre::class,
	'choice_label' => 'name',
	'multiple' => true,
]);
```

### Important Options

**- - > multiple = true**

This means: The user can select more than one genre ( `<select multiple>`).
Without it, only one selection is allowed (normal select).

**- - > Display as CHECKBOXES**

If you want checkboxes instead of a `<select multiple>`:

```php
$builder->add('genres', EntityType::class, [
 'class' => Genre::class,
 'choice_label' => 'name',
 'multiple' => true,
 'expanded' => true,
]);
```

**- - > Display as RADIO BUTTONS**

If you want radio button:

```php
$builder->add('genres', EntityType::class, [
 'class' => Genre::class,
 'choice_label' => 'name',
 'multiple' => false, # <==
 'expanded' => true,
]);
```

**- - > What does expanded = true do?**

- `false` → `<select multiple>`
- `true` → checkboxes

### When the Form is Submitted :

Symfony automatically:

- Converts selected IDs
- Fetches corresponding Genre objects
- Updates the `$book->genres` collection
- Manages the join table (`book_genre`)

You don’t manually insert into the join table.

## Executing SQL Queries in Symfony (Doctrine)

To create complex queries, we define custom methods inside the **Repository** class.

There are three main ways to write queries in Doctrine:

1.  **QueryBuilder** (recommended when possible)
2.  **DQL** (Doctrine Query Language)
3.  **Native SQL** queries

### 1) QueryBuilder (Recommended)

QueryBuilder allows you to build object-oriented queries using the Builder pattern.  
It is especially suitable for dynamic queries (conditional filters, optional parameters, etc.).

## Examples:

### 1. Sub-queries (Nested Queries)

**Example: Retrieve books whose genre is NOT Science Fiction**

```php
public function getBooksAreNot(Genre $genre): array
{
    $queryBuilder = $this->createQueryBuilder(alias: 'book');

    // Select all books with the genre = the given genre
    // These books will then be excluded
    $booksToExclude = $this->createQueryBuilder(alias: 'sub_books_query')
        ->select(...select: 'sub_books_query.id') // select the books' ids only
        ->join(join: 'sub_books_query.genres', alias: 'genres')
        ->where(...predicates: 'genres = :genre');

    return $queryBuilder
        ->where($queryBuilder->expr()->notIn(x: 'book.id', $booksToExclude->getDQL()))
        ->setParameter(key: 'genre', $genre)
        ->getQuery()
        ->getResult();
}
```

> 💡 A **sub-query** is built separately using a second `createQueryBuilder()`. It fetches the IDs of books that **belong** to the given genre. The main query then uses `notIn()` to **exclude** those IDs — returning only books that do NOT belong to that genre.

### 2. Filtering by Multiple Genres (IN clause)

**Example: Display Science Fiction OR Fantasy books**

```php
public function getBooksFilteredByGenres(array $genres): array
{
    $queryBuilder = $this->createQueryBuilder(alias: 'books');

    return $queryBuilder
        ->select(...select: 'books')
        ->join(join: 'books.genres', alias: 'genres')
        ->where($queryBuilder->expr()->in(x: 'genres', y: ':genres'))
        ->setParameter(key: 'genres', $genres)
        ->getQuery()
        ->getResult();
}
```

> 💡 The `in()` expression checks if the book's genre is **within an array** of genres passed as a parameter. This is the equivalent of SQL's `WHERE genre IN (...)`.

### 3. Search by ID or ISBN

**Example: Find a book by its ID or ISBN number**

```php
public function getBookByStandardNumberOrId(string $identifier): ?Book
{
    $queryBuilder = $this->createQueryBuilder(alias: 'book');

    return $queryBuilder
        ->where(...predicates: 'book.id = :id')
        ->orWhere(...where: 'book.isbn = :isbn')
        ->setParameter(key: 'id', $identifier)
        ->setParameter(key: 'isbn', $identifier)
        ->getQuery()
        ->getOneOrNullResult();
}
```

> 💡 The same `$identifier` value is used for **both** parameters. The query tries to match either the `id` or the `isbn` field. `getOneOrNullResult()` returns the book or `null` if nothing is found.

### 4. Search by Title or Summary (Partial String Match)

**Example: Find books whose title or summary contains a given string**

```php
public function findBooks(?string $word): array
{
    $queryBuilder = $this->createQueryBuilder(alias: 'book');

    if (\is_string($word)) {
        $expr = $queryBuilder->expr();

        $queryBuilder
            ->where($expr->like(x: 'book.title', y: ':title'))
            ->orWhere($expr->like(x: 'book.summary', y: ':summary'))
            ->setParameter(key: 'title', value: '%' . $word . '%')
            ->setParameter(key: 'summary', value: '%' . $word . '%');
    }

    return $queryBuilder
        ->getQuery()
        ->getResult();
}
```

> 💡 The `%` wildcards around `$word` allow a **partial match** anywhere in the string (equivalent to SQL `LIKE '%word%'`). If no word is provided, **all books** are returned.

### 5. Filter by Number of Pages (Conditional Clauses)

**Example: Retrieve books filtered by a min and/or max page count**

```php
public function getBooks(?int $minPages = null, ?int $maxPages = null): array
{
    $queryBuilder = $this->createQueryBuilder(alias: 'book');

    if (null !== $minPages) {
        $queryBuilder
            ->andWhere(...where: 'book.nbPages >= :minPages')
            ->setParameter(key: 'minPages', $minPages);
    }

    if (null !== $maxPages) {
        $queryBuilder
            ->andWhere(...where: 'book.nbPages <= :maxPages')
            ->setParameter(key: 'maxPages', $maxPages);
    }

    return $queryBuilder->getQuery()->getResult();
}
```

> 💡 Each condition is **optional** — filters are only applied if the parameter is not `null`. This makes the method flexible: you can filter by min only, max only, both, or neither.

### 6. Search Free Books by Title or Description

**Example: Find books whose title or summary contains a string AND whose price is 0 (free)**

```php
public function searchFreeBooksPartially(string $word): array
{
    $queryBuilder = $this->createQueryBuilder(alias: 'book');

    $expr = $queryBuilder->expr();

    $condition = $expr->andX(
        $expr->orX(
            $expr->like(x: 'book.title', y: ':title'),
            $expr->like(x: 'book.summary', y: ':summary'),
        ),
        $expr->eq(x: 'book.price', y: 0)
    );

    return $queryBuilder
        ->where($condition)
        ->setParameter(key: 'title', value: '%' . $word . '%')
        ->setParameter(key: 'summary', value: '%' . $word . '%')
        ->getQuery()
        ->getResult();
}
```

> 💡 `andX()` and `orX()` let you **combine multiple conditions** programmatically. The logic here is: `(title LIKE x OR summary LIKE x) AND price = 0` This is cleaner and safer than chaining raw DQL strings.

### 7. Books NOT Belonging to a Genre (Sub-query with Inclusive Option)

**Example: Retrieve all books whose genre list does NOT contain the given genre**

```php
public function getBooksDontBelongTo(Genre $genre, bool $inclusive = false): iterable
{
    if (true === $inclusive) {
        return $genre->getBooks();
    }

    $queryBuilder = $this->createQueryBuilder(alias: 'b');

    $subQuery = $this->createQueryBuilder(alias: 'sb');

    // Select only the IDs of books that belong to the genre
    $booksToExclude = $subQuery
        ->select(...select: 'sb.id')
        ->join(join: 'sb.genres', alias: 'g')
        ->where(...predicates: 'g = :genre');

    return $queryBuilder
        ->where($queryBuilder->expr()->notIn(x: 'b.id', $booksToExclude->getDQL()))
        ->setParameter(key: 'genre', $genre)
        ->getQuery()
        ->getResult();
}
```

> 💡 The `$inclusive` flag acts as a **shortcut**: if `true`, it simply returns the genre's own books directly. If `false`, a sub-query finds books that **belong** to the genre, then the main query **excludes** them using `notIn()`.

### 8. Choosing the Right Result Method

Picking the wrong result method is one of the **most common bugs** with the Query Builder. Always ask yourself: _"How many rows do I expect back?"_

### - - > If the query can return **multiple records**, use one of:

```php
$queryBuilder->getQuery()->getResult();       // Returns an array of objects (entities)
$queryBuilder->getQuery()->getArrayResult();  // Returns an array of plain arrays
```

---

### - - > If the query returns **exactly one record**, use one of:

```php
$queryBuilder->getQuery()->getSingleResult();     // ⚠️ Throws an exception if not found
$queryBuilder->getQuery()->getOneOrNullResult();  // ✅ Returns null if not found (safer)
```

---

### - - > If the query returns a **single scalar number** (e.g. `COUNT`, `SUM`):

```php
$queryBuilder->getQuery()->getSingleScalarResult();
```

---

### - - > If the query returns a **flat list of scalar values** (e.g. result of `GROUP BY`):

```php
$queryBuilder->getQuery()->getScalarResult();
```

---

> 💡 **Quick Summary Table**
>
> Situation -> Method to use
> +++++++\_\_\_+++++++++++
> Multiple entities expected -> `getResult()`
> Multiple raw arrays expected -> `getArrayResult()`
> Exactly one entity (throws if missing) ->`getSingleResult()`
> One entity or nothing -> `getOneOrNullResult()`
> One number (COUNT, SUM, etc.) -> `getSingleScalarResult()`
> Flat list of scalar values -> `getScalarResult()`

---

### 2) DQL (Doctrine Query Language)

DQL is similar to SQL, but it operates on entities instead of tables.

- Table names → replaced by entity class names
- Column names → replaced by entity property names

**- - > Example: Books ordered by price**

```php
public function findAllOrderedByPrice(): array
{
	return $this->getEntityManager()
		->createQuery('SELECT b FROM App\Entity\Book b ORDER BY b.price ASC')
		->getResult();
}
```

**- - > Example: Search by keyword**

```php
public function searchByKeyword(string $keyword): array
{
	return $this->getEntityManager()
		->createQuery('
		SELECT b FROM App\Entity\Book b
		WHERE b.title LIKE :k OR b.summary LIKE :k
		')
		->setParameter('k', '%' . $keyword . '%')
		->getResult();
}
```

DQL is readable and expressive, but less convenient for dynamic queries compared to QueryBuilder.
Tip: You can get the DQL from QueryBuilder: `$queryBuilder->getQuery()->getDQL();`

---

### 3) Native SQL

If a query cannot easily be expressed in QueryBuilder or DQL, you may execute raw SQL.

However:

- The result is returned as plain arrays
- No entity hydration occurs

**- - > Example:**

```php
public function findWithNativeSql(): array
{
	$conn = $this->getEntityManager()->getConnection();
	$sql = 'SELECT * FROM book WHERE price = 0';
	return $conn->executeQuery($sql)->fetchAllAssociative();
}
```

Use native SQL only when necessary.

## PHP Enumerations (Enums) in Symfony & Doctrine

### What is an Enumeration?

An **Enumeration (Enum)** is a special type that defines a fixed list of possible values. Think of it as a set of named constants grouped together under one type.

### - - > Common Use Cases

- When you want to represent a **status** (e.g. `shipped`, `canceled`, `pending`)
- When you want to enforce a **fixed list of choices** (like a dropdown)
- Anywhere you want to **restrict a value** to a known, controlled set

---

### 1. Defining an Enum in PHP

**Example: A book can be either `New` or `Used` (Bonne occasion)**

```php
enum BookStateEnum: string
{
    case NEW = 'new';
    case USED = 'used';

    public function label(): string
    {
        return match ($this) {
            self::NEW  => 'Nouveau',
            self::USED => 'Bonne occasion',
        };
    }
}
```

> 💡 PHP enums can be **backed by a type** (here `string`). Each `case` maps to a string value stored in the database. The `label()` method returns a **human-readable** version of each case — useful for displaying in forms or templates.

---

### 2. Using an Enum in a Doctrine Entity

**Example**: Mapping the `$status` property to the `BookStateEnum` enum in Doctrine

```php
use App\Enum\BookStateEnum;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Column(enumType: BookStateEnum::class)]
private ?BookStateEnum $status = null;


public function getStatus(): ?BookStateEnum
{
    return $this->status;
}

public function setStatus(BookStateEnum $status): static
{
    $this->status = $status;

    return $this;
}
```

> 💡 The `#[ORM\Column(enumType: BookStateEnum::class)]` attribute tells Doctrine to **automatically convert** the database string value back into the proper Enum case when reading, and to store the Enum's string value when writing. This means you work with **type-safe enum objects** in your PHP code instead of raw strings.

---

### 3. Using an Enum in a Symfony Form

**Example**: Rendering the enum as a dropdown (`EnumType`) in a Symfony form

```php
use App\Enum\BookStateEnum;
use Symfony\Component\Form\Extension\Core\Type\EnumType;

class BookType extends AbstractType
{
    public function buildForm(
        FormBuilderInterface $builder,
        array $options,
    ): void {
        $builder
            // ...
            ->add('state', EnumType::class, [
                'class'        => BookStateEnum::class,
                'choice_label' => fn (BookStateEnum $state) => $state->label(),
            ])
        ;
    }
	//...
}

```

> 💡 `EnumType` works like a standard `ChoiceType` (dropdown), but it **automatically reads all cases** from your enum class. The `choice_label` option uses the `label()` method we defined earlier to display friendly names instead of raw values like `'new'` or `'used'`.

### Summary

Steps to use enums:

**1. Create the Enum** Define cases and optionally a `label()` method for display

**2. Map it in the Entity** Use `#[ORM\Column(enumType: YourEnum::class)]`

**3. Use it in the Form** Add the field with `EnumType::class` and a `choice_label` callback
