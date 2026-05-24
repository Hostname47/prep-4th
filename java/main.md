# Java Compete course

## dependency relations types

**A ─────▶ B** (A uses B)

- **A ◇──── B** (aggregation)

- **A ◆──── B** (composition)

**A - - - -▷ B** (A implement an interface B)

**A ─────▷ B** (A inherit from B)

## Usage (Association)

Represents a relationship where one class **uses or depends on** another : **A ─────▶ B**

in usage, we have two types:

### Aggregation (Weak "has-a") => A ◇──── B

- A form of usage where one class **contains** another.
- The contained object can **exist independently**.
- Example: A `Team` has `Players`, but players can exist without the team.

---

### Composition (Strong "has-a") => A ◆──── B

- A stronger form of aggregation.
- The contained object **cannot exist without** the container.
- If the container is destroyed, so are its parts.
- Example: A `House` has `Rooms`; rooms don’t exist without the house.

So to distinguish between the weak and strong forms, we ask the question: if we remove the parent object, does the child remains ort not ? if yes, then we have aggregation, if no, we have composition.

### 2. Implementation (A implements B)

A - - - -▷ B

### 3. Inheritance (A inherits from B)

A ─────▷ B

Here we have the verb to be (Employee IS A person) => Inheritence

---

In order to know if there is a relationship bertween two classes, you need always to understand the functional semantic between the two entites that will help you determin the type of relationship and also if you have weak or strong, or if you have inheritance.

So once you have a relation usage or inheritance we say we have a coupling; coupling means you have a link between entities, and it can be a strong coupling or weak coupling.

It is prefferable to not have any coupling between entities, but in the real world, it is always a must to have coupling.

Coupling: when you have A depends on B, can we compile A without havingh B => NO so we have a coupling and we need B first before compiuling A.
But notice that we can compile B without having A. Also B can be compiled and used independent of A. (Same rule applied in inheritance)

A inheit from B, so we need B first before compiling A, but we can use B without A and independent of A

In UML, there is a difference in diagrams and arrows:

1. Inheritance (Generalization)
   A ─────────▷ B

2. Interface Implementation (Realization)
   A - - - - -▷ I

3. Usage (Dependency)
   A - - - - -> B

4. Association
   A ---------> B

5. Composition
   A ◆-------- B

6. Aggregation
   A ◇-------- B

between a class and interfgaqce, it could be usage or it could be implementation

between interfaces, it could be inheritence; an interface can inherit from other interface; an interface cannot inherit fronm a class; a class can implement an interface; a class cannot inherit from an interface.

My research (not included in cours): Between two interfaces, it could be also a usage relationship

Also, a class can **use** an interface (has a prop or an attr that is typed with this interface)

---

**CLASS → CLASS RELATIONSHIPS**

Inheritance:
A ─────────▷ B

Dependency (Usage):
A - - - - -> B

Association:
A ---------> B

Aggregation:
A ◇-------- B

Composition:
A ◆-------- B

---

**CLASS → INTERFACE RELATIONSHIPS**

Implementation (Realization):
A - - - - -▷ B

Dependency (Usage):
A - - - - -> B

Association:
A ---------> B

---

### Differentiate between weak and strong coupling

#### Strong Coupling

- `Order` directly depends on a concrete writer. (pay attention to new ketyword used [strong coupling])

**Example:**

```java
class OrderStateXMLWriter {
    void write(String state) {}
}

class Order {
    String getState() { return "CREATED"; }

    void write() {
        OrderStateXMLWriter writer = new OrderStateXMLWriter();
        writer.write(this.getState());
    }
}

```

**Diagram:** Order ─────▶ OrderStateXMLWriter (❌ Hard to replace writer implementation)

---

#### Weak Coupling

- `Order` depends on an interface.

**Example:**

```java
interface OrderStateWriter {
    void write(String state);
}

class OrderStateXMLWriter implements OrderStateWriter {
    public void write(String state) {}
}

class OrderStateDBWriter implements OrderStateWriter {
    public void write(String state) {}
}

class Order {
    private OrderStateWriter writer;

    Order(OrderStateWriter writer) {
        this.writer = writer;
    }

    String getState() { return "CREATED"; }

    void write() {
        writer.write(this.getState());
    }
}
```

**Diagram:**
Order ─────▶ OrderStateWriter  
OrderStateXMLWriter - - - -▷ OrderStateWriter

- ✅ Easy to switch writer (XML, JSON, etc.)

---

## Quick Tip

- Pass **data (state)**, not responsibility → keeps design flexible.

#### Class diagram steps to create

1. Identify only classes and write them down
2. Take each class, and define its attributes and methods
3. Identify relationships between classes (usage, inheritance,...)

in usage relationship, when A uses B, it could be one of the following 3 cases: B is a type of a prop in A, or inside a method partameters of A,or as a local variables,...

If A uses B, and then we change someway the signature of a method in B that A already uses, we will face issues and A cannot be compiled (regression), but B itself is fine and able to be compiled !

**Symptoms of strong coupling:**

1. The first symptom of strong coupling is regression that happens when we change in B a method signature that A already uses, or change a type or remove a method, A fails and stop working.

2. The second symptom: In A, instead of using B, we decide to change the strategy to a class B' which does the same job as B but in a different way, and this B' can have the same emthod that does the job we want, but it may has a different name of the method we already had in B, and because A alread has calls to method of B, and we change B with B' so A fails, now we need to **do changes in A** and adjust it with B' in order to make it work again. So Changing B to B' causes A to fails ===> We have a strong coupling. So once we change class B with B' that does the same job forces us to make changes in A, this means WE HAVE STRONG COUPLING.

So the two saymptoms in brief: If changing B's method signature or structure cause A to complain we say we have strong coupling; If replacing B with another class B' that does the same job as B but differently, and we find ourselves obkligated to make changes in A, here also we have strong coupling.

#### HOW TO AVOID STRONG COUPLING

To break strong coupling, we introduce an interface between A and B; Interface represente a contract between A and B, so instead of A uses B directly, it wait for an instance of interface, and B needs to implement I in order to be uised in A. So B needs to adhere to I contract.

The goal of then interface here is to frame how B should be in order for A to use it, so now B needs to respect and implement the contract that I implicate literally, and when A uses B through I, now we will not have any worries that if B change its methods names or signature and cause A to fail, this will never happen since the contract obligate for B to respect literally with all contracts and rules.

Also if we have a B', we can also to implement the same interface and A will switch the strategy very easily and does not need to adjust how it uses its dependency and its methods how they called or what is their signatures..

In software engineering, interfaces are the pillars of programming, like HTTP protocol: is a set of rules and contracts to be respected: the fi4rst bytes should have url, then we have destination IP, then....

so http protocol is independent of browser and will construct the http request in the same way in all berowsers. Now in our example, A does not know B it knows I; I propose obligations foir classes that want to implement it to implement a set of contracts and rules. Classes that implements an interface need to implement LITERRALYY and in total of all rules implicated by thhe interface.

So to **escape from strong coupling**, and our goal is to frame the way classes that A need should be, and the way to have this is to make A independent of its classes B and B' and make A depend on abstraction [Interface | Contract];

Interface represents an abstraction => the main important rule of this abstraction: IT SHOULD COVER ALL POSSIBLE CASES.. that A wait from B or B'; The way we get those cases and functionalities is to look at the detailed specs, or technical detailed specs, functional needs (cahier de charges)... Please notice that we need only to cover cases that are present in sepcifications.

Le Document d'Architecture Technique (DAT) est un dossier de référence crucial décrivant l'ensemble des composants techniques (matériels, logiciels, réseaux, sécurité) d'un système informatique. Il aligne les choix techniques avec les besoins fonctionnels, assurant la performance, la sécurité et la maintenabilité, rédigé généralement par un architecte technique.

After defining and setting an interface in between A and its dependencies, now if B decide to change the signature of a method that A already use and within the interface, then it will get an error, that obligate oit to implement the interface coirrectly; So the interface now will be a source of truth and make sure that A will always be stable (Chnages in B, will cause compilation errors in B) => so the first symmprom of strong coupling is solved.

Also, if B' has different name of methods or signature other than the one in interface, here also we will have compilation errors in B',, and so B' needs to adhere to I rules and signatures in order to be used in A, so the second symtom also get solved with weak coupling.

Contract is the important part of this flow; because they represent the foundation of your system, and making changes on them, resuylots in changes on all implementations and also in places where you use instances that implement them. And because of that, THE MOST DIFFICULT PART, is to define your contracts, that represent the most of cases on your system.

Please pay attention to contracts and give them their importance and craft them with care.
