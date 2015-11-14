---
comments: true
layout: post
title:  "Slick 1.0 - How I deal with relations"
date:   2013-09-18 18:09:01
categories: programming scala slick
---
In this post I want to show you on example how I deal with relations in Slick. I don't pretend that it's the best or the most elegant solution, so feel free to leave your comments :). Before we dive in to the relations, I want to start with an observation of some basics.

As you know Slick is not an ORM but FRM and it means Functional-Relational Mapping and that determines a philosophy behind him.

So let's imagine that we have two entities -  Product and Supplier then we create classes that will represent them. Our code could look like this :

```scala
case class Product(id : Option[Int] = None, ref : String, label : String, price : Double, fk_supplier : Option[Int])
case class Supplier(id : Option[Int] = None, name : String)
```

Now that we have our classes for data, we need to create the classes that will represent our database tables and provide us with possibility to persist and to fetch our data, one important thing to notice here is that it's better to define *"Tables"* as *classes* not as a *singleton objects* and this will avoid problems with some slick features in the future. So for the Product:

```scala
class ProductTable extends Table[Product]("t_product"){
    def id = column[Option[Int]]("id", O.PrimaryKey, O.AutoInc)
    def ref = column[String]("name"),
    def label = column[String]("label"),
    def price = column[Double]("price"),
    def fk_supplier = column[Option[Int]]("fk_supplier")
    def * = id ~ ref ~ label ~ price ~ fk_supplier <> (Product.apply _, Product.unapply _)
    def autoInc = * returning id
}
```
and for the Supplier :

```scala
class SupplierTable extends Table[Supplier]("t_supplier"){
  def id = column[Option[Int]]("id", O.PrimaryKey, O.AutoInc)
  def name = column[String]("name")
  def * = id ~ name  <> (Supplier.apply _, Supplier.unapply _)
}
```

In our example we use Lifted embedding and this means that we are not working with standard Scala types but with types that are lifted into the scala.slick.lifted.Rep type constructor. In query `String` becomes `Rep[String]`, `Double` becomes `Rep[Double]` and so on. You may get more details about Lifted Embedding from Slick documentation here. From the source code we can see that we define columns through the column method
and we provide Scala type `Option[Int]` and a column name "id" for it. After the column name we can specify some extra options which are provided for us by the table object in our case *O object* and are used for DDL creation. So in the code above we use `O.PrimaryKey` that marks the column as a (non-compound) primary key and `O.AutoInc` that marks it as an auto-incrementing key. At the end of our Table object we define method that is required by every table object and contains a default projection - that is what we get back when we return rows from the query. In our case we use mapped tables which are mapped to the case classes via bi-directional mapping in projection with `<>` operator .

But you are free to create your tables with out mapping, using Tuples for that and your code may be like this:

```scala
class Address extends Table[(Option[Int],String,String,String)]("t_address"){
    def id = column[Option[Int]]("id", O.PrimaryKey, O.AutoInc)
    def address = column[String]("address")
    def city = column[String]("city")
    def zip = column[String]("zip")
    def * = id ~ address ~ city ~ zip
}
```
and you will get back tuple while querying.

We have classes and tables defined, okay let's create some queries. Create queries in Slick is pretty straightforward, we deal with them absolutely  like with Scala collections. Consider the following code to fetch Supplier:

```scala
def getById(id : Int) : Supplier = {
    (for{s<-new SupplierTable if(s.id === id)}yield s).first
}
```

We use **"for comprehension"** here to iterate over the suppliers with "if condition" to yield supplier with required id and at the end we apply "first" method to extract `Supplier` from `Query[SupplierTable.type,Supplier]`. Another way to write this query is to use a filter on Query object:

```scala
def getById(id : Int) : Supplier = {
    Query( new SupplierTable).filter(_.id === id).first
}
```

which will produce exactly the same result - Supplier object with required id.

To "insert" our data to the database we have methods "insert" and "insertAll"

```scala
val supplierTable = new SupplierTable
val productTable = new ProductTable
supplierTable.insert(Supplier(None,"Supplier 1"))
productTable.insertAll(
    Product(None,"Product 1","", Some(1)),
    Product(None,"Product 2","", Some(1)),
    Product(None,"Product 3","", Some(1)),
    Product(None,"Product 4","", Some(1))
)
```

To update we use "update" method for example we can define update method for our Product in this way:

```scala
def updateProduct(p : Product) = {
    productTable.were(_.id===p.id).update(p)
}
```

For "delete" we apply exactly the same logic:

```scala
def deleteProduct(p : Product) = {
    productTable.were(_.id===p.id).delete
}
```

So far, so good but what if we want to have one to many relation between Supplier and Product and while fetching a Supplier also get a list of all his products.

First of all let's add a foreign key constraint to our ProducTable, we do it with foreignKey method,

```scala
def supplier = foreignKey("fk_supplier",fk_supplier,SupplierTable)(_.id,onDelete = ForeignKeyAction.Cascade)
```

it takes four parameters : "name of the constraint", "column to applic to", "table to link to" and a function from that table to corresponding column. Then we join supplier to the product and products to supplier

```scala
def supplierJoin = (new SupplierTable).where(_.id===fk_supplier)
def productsJoin = Query(new ProductTable).where(_.fk_supplier === id)
```
Now our ProductTable and SupplierTable is as follows:

```scala
class ProductTable extends Table[Product]("t_product") {
    def id = column[Option[Int]]("id", O.PrimaryKey, O.AutoInc)
    def ref = column[String]("name"),
    def label = column[String]("label"),
    def price = column[Double]("label"),
    def fk_supplier = column[Option[Int]]("fk_supplier")
    def supplier = foreignKey("FK_SUPPLIER",fk_supplier,SupplierTable)(_.id)
    def supplierJoin = (new SupplierTable).where(_.id===fk_supplier)
    def * = id ~ ref ~ label ~ price ~ fk_supplier <> (Product.apply _, Product.unapply _)
    def autoInc = * returning id
    /* get product by ID */
    def getByID(id : Int) : Product = Query(new ProductTable).filter(_.id === id).first
}
class SupplierTable extends Table[Supplier]("t_supplier") {
    def id = column[Option[Int]]("id", O.PrimaryKey, O.AutoInc)
    def name = column[String]("name")
    def * = id ~ name  <> (Supplier.apply _, Supplier.unapply _)
    def autoInc = * returnig id
    def productsJoin = Query(new ProductTable).where(_.fk_supplier === id)
    
    /* get supplier by ID */
    def getById(id : Int) : Supplier = Query(new SupplierTable).filter(_.id===id).first
}
```

and in Supplier class we can define method like this: 

```scala
def products : List[Product] =(
    for{
        s <-Query(supplierTable) if(s.id === id)
        p <- s.productsJoin 
    } yield p
).list
```
Now for example while querying for a Supplier

```scala
val supplier = SupplierTable.getById(1)
val products = supplier.products // val products contains  list of all products for this supplier
```

Having this in mind let's complicate the case. What if one and the same product can have several suppliers? Right, we have to deal with "many to many" relations. To implement this we will need to introduce intermediate table that will hold our  products `<->` suppliers relations.

We have two foreign keys here, one for product and for supplier(and no longer need to keep foreign key for supplier in a ProductTable). Then join suppliers to product and products to supplier

```scala
def suppliersJoin = Query(ProductsToSuppliers).filter(_.prodId===id).flatMap(f=>f.fk_sup)

def productsJoin = Query(ProductsToSuppliers).filter(_.suppId === id).flatMap(p=>p.fk_prod)
```

You may have noticed that we have used our foreign keys methods to perform joins.

Ok, it's time to look how we will use this to get all suppliers naturally when querying for a product. Let's define method "suppliers" in the Product class that will do it for us:

```scala
def suppliers = (
    for{
        p <- Query(productTable).filter(_.id === id)
        s <- p.suppliers
    } yield s
).list
```

That is, and making a query for a product we are able to get the list of the suppliers exactly as if Product have a property "suppliers : `List[Supplier]`". The same is almost true for the Supplier

```scala
val product = ProductTable.getById(1)
val suppliers = product.suppliers // val suppliers contains  list of all suppliers for this product
```
Summing up, we have briefly passed through the basics of slick (table definition, simple queries) and considered one of the ways to deal with relationships.