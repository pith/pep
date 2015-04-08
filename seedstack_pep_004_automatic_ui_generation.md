
- PEP: 004
- Title: Automatic UI generation (codenamed Kaviar)
- Authors: 
 - Adrien LAUER <adrien.lauer@mpsa.com>
 - Kavi RAMYEAD <kavi.ramyead@ext.mpsa.com>
 - Pierre THIROUIN <pierre.thirouin@ext.mpsa.com>
 - Redouane LOULOU <redouane.loulou@ext.mpsa.com>
- Status: draft

# Abstract

Implements features needed to automatically expose representations. We already have automatic repository and factory. We now need:
- Automatic assembler
- Automatic finder
- Resource templates
- Hypermedia links between representations
- Standard UI patterns

# Motivation

We want the business framework to be a complete productivity framework based on DDD principles. The creation of DDD building blocks has already been addressed, so in this PEP we focus on speeding up development with the business framework.

# Specification


## Assemblers

This part is located in business framework core.

The first step leading to automatic UI will be to provide automatic
assemblers. So users won't have to write them, even if it will be still configurable.

In order to do this, we will create a specific assembler extending
`AbstractBaseAssembler`. It will use the DSL provided by [ModelMapper](http://modelmapper.org/).

**TODO** this draft needs some complements on:

* POC with maps and lists
* POC with tuples
* Data security

### Automatic assembler

```java
public class AutomaticAssembler<AGGREGATE_ROOT extends AggregateRoot<?>, DTO extends Object> extends AbstractBaseAssembler<AGGREGATE_ROOT, DTO>{
...
}
```

`AutomaticAssembler` will keep the compatibility with the current
assemblers by specializing them for automatic mapping. The do...()
methods will already be implemented in those automatic assemblers with
the ability for end users to override a configure() method if special
configuration is needed. An explicitly written assembler class will
not be needed if the developer want to keep the default behavior (much
like the default factories or repositories). In that case, an
annotation must be applied on the DTO class for the framework to
recognize the association to the aggregate. **This part will
supplement the currently existing assemblers.**

```java
public MyAssembler extends AutomaticAssembler<MyAggregateRoot, MyDTO> {

    public void configureAssembly(MyAggregateRoot source) {
        map().getAddress().setStreet(source.getStreet());
        skip(source.name);
    }

    public void configureMerge(MyDTO source) {
        map().getAddress().setStreet(source.getStreet());
    }

}
```

Note that map(), skip() and other methods should detect from which
method it is called to provide the right target.

### Assembler DSL

On top of that a DSL will be provided to developers to specify
the whole assembling behavior, including loading from a
repo, creating from a factory and so on... **This part will
completely replace the current Assemblers facade, which should be
deprecated**.

** Sequence diagrams **

![Sequence diagram dto to aggregate root](./seedstack_pep_004_automatic_ui_generation/dslToAgg.png)

![Sequence diagram aggregate root to dto](./seedstack_pep_004_automatic_ui_generation/dslToDto.png)

**Dto to aggregate**

```java
import static org.seedstack.business.api.interfaces.Interfaces.assemble;
...

OrderDto orderDto = new OrderDto();
Order myOrder = new Order();
Customer customer = new Customer();
List<Object> dtos = Lists.newArrayList(orderDto, myOrder);
List<Order> orders = Lists.newArrayList(myOrder, myOrder);

Order order = assemble().dto(orderDto).to(myOrder);

order = Interfaces.assemble().dto(orderDto).to(Order.class).fromFactory(); // from factory

order = assemble().dtos(dtos).to(Tuple.tuple(Order.class, Customer.class)).fromFactory(); // list of dto to tuple of aggregates

order = assemble().dtos(dtos).to(orders); // list of dtos to list of aggregates

order = assemble().dto(orderDto).to(Order.class).fromRepository().thenFromFactory(); // from repo or fact
try {
    order = assemble().dto(orderDto).to(Order.class).fromRepository().orFail(); // from repo or fail
} catch (AggregateNotFoundException e) {
    e.printStackTrace();
}
```

**With qualifier**

```java
order = assemble().dto(orderDto).to(Order.class)
    .fromRepositories(Names.named("jpa"), Names.named("jdbc"))
    .thenFromFactories(Names.named("fact1"), Names.named("fact2"));
```

**Aggregate to dto**

```java
OrderDto orderDto1 = assemble().aggregate(myOrder).to(OrderDto.class);

orderDto1 = assemble().aggregate(myOrder).toDynamicDto();

orderDto1 = assemble().aggregates(Lists.newArrayList(myOrder1, myOrder2)).toDynamicDto();
```

**Tuple of aggregate to dto**

```java
orderDto1 = assemble().tuple(Tuple.tuple(Order.class, Customer.class)).to(OrderDto.class);

orderDto1 = assemble().tuples(
        Lists.newArrayList(Tuple.tuple(order, customer), Tuple.tuple(order, customer))
).to(OrderDto.class);
```

### Matching DTO parameters to factory's methods

In order to implement the methods `fromRepository()` and
`fromFactory()` we need:
1. load an aggregate from a repository based its id 
2. Create an aggregate with a factory based on the dto

For the first case, we need to find/construct the aggregate id from the dto.
For the second case, we need to map dto getter methods to the factory
method parameters.

What we have now.

```java
public class ProductDto {
    private Short storeId;
    private Short productCode;
    private String name;
    private Integer price;

    @MatchingFactoryParameter(index=0)
    public String getName() {
        return name;
    }

    @MatchingAggregateId (index=0)
    @MatchingFactoryParameter(index=1)
    public Short getStoreId() {
        return storeId;
    }

    @MatchingAggregateId (index=1)
    @MatchingFactoryParameter(index=2)
    public Short getProductCode() {
        return productCode;
    }
}
```

```java
@Embeddable
public class ProductId extends BaseValueObject {
    private Short storeId;
    private Short productCode;

    ProductId() {
    }

    public ProductId(Short storeId, Short productCode) {
        this.storeId = storeId;
        this.productCode = productCode;
    }
}
```

```java
public interface ProductFactory extends GenericFactory<Product> {
    Product createProduct (String name, Short storeId, Short productCode);
}
```

We may reduce the annotation `@MatchingAggregateId` to `@MatchId`
and `@MatchingFactoryParameter` to `@MatchParam`. And replace index by
value. This will improve the lisibility.

Also we plan to add different matching algorithms. Projects using Java
7 will benefit from a matching based on the method parameters's name.
People using `AutomaticAssembler`s will able to specify the matching
by themselves with modelmapper.

These different matching will be grouped under an SPI composed of one
interface:

```java
interface DtoInfo {

    Object resolveId(Object dto);

    Object resolveAggregate(Object dto);

}
```
## Finder improvements

This part is located in business framework core.

Advanced finders often requires multiple features like pagination, filtering or sorting. One should be able to compose such features for each specific finder. For instance, a finder could implement pagination and sorting but not filtering. An API based on a decorator pattern could provide this flexibility instead of inheritance (inability to compose) or interface composition (inability to implement default behavior).

A default finder implementation should be provided (like default factories, repositories, ...) which will have a simple but effective implementation of at least the three following features: pagination, filtering, sorting.

A DSL will enable to use the finders (whether a default or an explicit implementation) at a higher level of abstraction. Finder features will be easily addressable:

```
Find.chunk(15, 35).of(CustomerDTO.class).withFinder("qualifier1")
    .sortedBy("firstName").asc()
    .sortedBy("lastName")
    .filteredBy(theCriteria);

Find.page(5, 10).of(CustomerDTO.class)
    .sortedBy("firstName").asc()
    .sortedBy("lastName")
    .filteredBy(theCriteria);

Find.all().of(CustomerDTO.class);
```

## Resource templates

This part is located in business framework rest.

The idea is to allow the creation of resource templates which will be like normal resources but not automatically registered. Instead they can be exposed multiple times with different representations and various configurations.

```
Expose.representation(CustomerRepresentation.class).with(CRUDResourceTemplate.class).on("/customers");

Expose.dynamicRepresentationOf(Customer.class).with(CRUDResourceTemplate.class).on("/customers");

Expose.representation(CustomerRepresentation.class).with(CRUDResourceTemplate.class).on("/customers")
    .usingFactory("qualifier1")
    .usingRepository("qualifier2")
    .usingFinder("qualifier3")
    .usingAssembler("qualifier4");
```

## Hypermedia

This part is located in business framework rest.

## UI generation

### Screens

The front end part of the automatic UI generation function should provide default ready to use screens for each registered resources while allowing extensiblilty and customization. 

In order to achieve this goal, the proposed architecture is based on the idea of AngularJS directives composition to assemble a master screen. 

If we take as an example the 'master-detail' pattern, we can break the pattern into an assembly of several micro templates encapsulated in directives: a 'ListToolbar' template containing an 'Add' button, a 'MainList' of entries, an 'EntityDisplay' area for the details on the selected items, an 'EntityToolbar' at the bottom with a 'Delete' and 'Update' action and so on. These directives are combined to form a master template representing the final screen.

     Screen
        Directive List (list1)
          Directive ListToolbar (listToolbar1)
          Directive MainList (mainList1)
        Directive Entity (entity1)
          Directive EntityDisplay (entityDisplay1)
          Directive EntityForm (entityForm1)
          Directive EntityToolbar (entityToolbar1)
          
A screen has a type which represent the specific arangement of its directives. A screen can be customized by overiding the disposition of the directives in the html or by adding custom elements around the building blocks directives. A screen is associated with a controller which holds the most common scope ancestor for all the element in the view. This allows sibling directives to communicate through a higher level scope hierarchy.

Directives are uniquely named in the html. This is useful for allowing configuration and wiring between directives at the screen level. For instance a screen can provide a configuration facade which allows to specify binding between the list and the entity id:

    list1.mainList1.selected = 2
    
By uniquely naming directive we can allow multiple directives of the same type to coexist on the same page (we can add a second list inside the screen and still differentiate for instance).

Directive components can also regsister events for communicating in different contexts.

One drawback of the micro templating approach is that it seems hard to override a micro template because of the encapsulation inside a directives. While it is probably true, directives could be sufficiently open to allow customization programmatically (adding an action button to a toolbar for instance or allowing hooks inside code execution).  

### Services

TODO

## UI patterns

This part is located in the kaviar function.

UI patterns should allow developers to choose between common visual representations for building the automated web application.

We can identify the basic ones:

- Simple form : A unique form for unitary resource.
- Simple list : A list (or table) with search and pagination option. Can use modal window with the Simple form pattern for creating/editing. 
- Master detail: Two column view pattern with a list of entry and a detailled zone for the selected element.

These patterns could come as angular directives templates. These directives can then be used directly during the init phase to build the application or used manually by the developer if he needs to include the pattern inside a custom view.

## UI navigation

This part is located in the kaviar function.

The webapp needs to be aware of the available resources exposed for CRUD manipulation in order to provide corresponding navigation. This configuration can come as a resource exposed on a known url (ex: rest/config) or be build on the server. It should provide the name of the resource and the ui pattern type to use for this resource. Furthermore it could allow flags for enabling/disabling routes on the client or allow the developer to use a custom view instead of the default template if needed. This resource can also be used for registering the REST path into a service on the client.
