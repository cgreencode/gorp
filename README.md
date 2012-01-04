## Goals ##

* Support transactions
* Forward engineer db schema from structs (especially good for unit tests)
* Optional optimistic locking using a version column
* Pre/post insert/update/delete hooks
* Automatically generate insert/update statements for a struct
* Delete by primary key
* Select by primary key
* Optional trace sql logging

## Non-Goals ##

* Eliminate need to know SQL
* Graph loads or saves

## Examples ##

First define some types:

    type DbStruct {
        Id        int64,
        Created   int64,
        Updated   int64,
        Version   int32,
    }
    
    type Product struct {
        DbStruct
        Description  string,
        UnitPrice    int32,   // in pennies
        IsTaxable    bool
    }
    
    type Order struct {
        DbStruct
        PaymentType  string,
        IsPaid       bool,
        SalesTax     int32
    }
    
    type LineItem struct {
        DbStruct
        OrderId      int64,
        ProductId    int64,
        Quantity     int32,
        UnitPrice    int32
    }

Then create a mapper, typically you'd do this one time at app startup:

    // connect to db using standard Go exp/sql API
    // use whatever exp/sql driver you wish
    db, err := sql.Open("mysql", "myuser:mypassword@localhost:3306/dbname")
    
    // construct a gorp DbMap
    dbmap := &gorp.DbMap{Db: db}
    
    // register the structs you wish to use with gorp
    t1 : = dbmap.AddTable(Product{})
    t1.SetKeys(true, "Id")
    
    // or use the builder syntax
    dbmap.AddTable(Order{}).SetKeys("Id")
    
    // optionally override the table name
    dbmap.AddTableWithName(LineItem{}, "line_item").SetKeys("Id")

Then save some data:

    p1 := &Product{Description: "Wool socks", UnitPrice: 499, IsTaxable: true}
    p2 := &Product{Description: "Tofu", UnitPrice: 249, IsTaxable: false}
    
    // Pass as a pointer so that optional callback hooks
    // can operate on your data, not copies
    dbmap.Insert(&p1, &p2)
    
    // Because we called SetAutoIncrPK() on Product, the Id field
    // will be populated after the Insert() automatically
    fmt.Printf("p1.Id=%d\n", p1.Id)

You can also batch operations into a transaction:

    func SaveOrder(dbmap *DbMap, order *Order, items []LineItem) error {
        // Start a new transaction
        trans := dbmap.Begin()

        trans.Insert(order)
        for _, v := range(items) {
            trans.Insert(v)
        }

        // will commit or rollback transaction
        // if an error occurred at any point, a rollback
        // is performed, and the error returned here    
        //
        // if the commit is successful, a nil error is returned
        return trans.End()
    }
    
How would I set the date updated/created?

    // implement the PreInsert and PreUpdate hooks
    func (d *DbStruct) PreInsert(mapper *gorp.Mapper) error {
        d.Created = time.Nanoseconds()
        d.Updated = d.Created
        return nil
    }
    
    func (d *DbStruct) PreUpdate(mapper *gorp.Mapper) error {
        d.Updated = time.Nanoseconds()
        return nil
    }
    
How do you hook autogenerated primary keys?

    // implement the PostInsert hook
    func (d *DbStruct) PostInsert(mapper *gorp.Mapper) error {
        res, err := dbmap.RawQuery("select last_insert_id()")
        if res != nil {
            res.Next()
            res.Scan(&d.Id)
            res.Close()
        }
    }

What is Version used for?

    // Version is an auto-incremented number, managed by gorp
    // If this property is present on your struct, update
    // operations will be constrained
    //
    // For example:
    
    p1 := &Product{Description: "Wool socks", UnitPrice: 499, IsTaxable: true}
    dbmap.Save(p1)  // Version is now 1
    
    p2 := dbmap.Get(Product, p1.Id)
    p2.UnitPrice = 599
    dbmap.Save(p2)  // Version is now 2
    
    p1.UnitPrice = 399
    err := dbmap.Save(p1)  // Raises error - p1.Version == 1, which is stale
    if err != nil {
        // should reach this statement
        fmt.Printf("Got err: %v\n", err)
    }
    
