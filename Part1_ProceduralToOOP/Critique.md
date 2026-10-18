# Critique: Procedural Order System (C++)

I went through `order_system.cpp` before writing any C#. The program works and does what it says: customers, products, orders, paying, and total sales. The problem is how it's built. Below is every design problem I found, why I think it's a problem, and what can actually go wrong because of it.

---

## 1. One customer is split across 5 different arrays


int    customerIds[MAX_CUSTOMERS];
string customerNames[MAX_CUSTOMERS];
string customerEmails[MAX_CUSTOMERS];
string customerCities[MAX_CUSTOMERS];
bool   customerIsVip[MAX_CUSTOMERS];


A customer isn't a "thing or entity" in this code. It's 5 separate arrays, and the only thing connecting them is the index. Nothing in the code says that `customerNames[3]` and `customerEmails[3]` belong to the same person. It's just an agreement inside the programmer's head.

**What can go wrong:** say one customer has an email and another doesn't, and someone writes code that only fills `customerEmails` when there actually is an email. Now the emails array is one item shorter than the names array, every index after that point is shifted, and customer #4 ends up with customer #5's email. No error, no crash, just wrong data. Same thing happens if someone adds a "delete customer" feature and forgets to remove the item from one of the arrays.

The same problem applies to products (4 arrays) and orders (5 arrays + 2 two-dimensional arrays for the lines).

**What it should be:** all of a customer's data belongs together in one `Customer` object. Then it's impossible for the name to be at one position and the email at another.

---

## 2. Everything is global, so anyone can change anything

All the data lives in global variables, and any function can read or write any of it.

For example, `addLineToOrder` changes the product's stock directly:

productStock[productIndex] -= quantity;


The stock belongs to the product, but the product has no control over it. Any function anywhere can set it to `-50` and nothing will stop it.

**What can go wrong:**
- If the stock number is ever wrong, *any* function in the program could be the reason, so debugging means reading the whole file.
- You can't have two separate stores running at the same time. There's only one global copy of everything.
- You can't unit test it properly, because every test changes the same global state and affects the next test.

---

## 3. Relationships are stored as array positions, not real references

int orderCustomerIndexes[MAX_ORDERS];
int lineProductIndexes[MAX_ORDERS][MAX_LINES_PER_ORDER];

An order doesn't know *its customer*. It only knows "the customer that happens to be at position 2 in the arrays right now".

**What can go wrong:** this works today only because nothing is ever deleted or re-sorted. The day someone adds `deleteCustomer` or sorts the products by name, every order silently points to the wrong customer or the wrong product. Again, no error, just wrong invoices.

**What it should be:** an `Order` should hold a reference to the actual `Customer` object, and an `OrderLine` should hold a reference to the actual `Product`.

---

## 4. Old orders change their total over time

`calculateOrderTotal` recalculates the total from scratch every time it's called:


total += productPrices[productIndex] * lineQuantities[orderIndex][i];

if (customerIsVip[customerIndex])
    total = total * 0.90;


It uses the product's price **now**, not the price when the customer bought it. Same thing with VIP: it checks whether the customer is VIP **now**, not when the order was made.

**What can go wrong:** if the keyboard price goes up tomorrow, or a customer becomes VIP next month, the total of an order that was **already paid** changes. `totalSalesPaidOnly()` would then report a sales number that's different from the money that was actually collected. For anything involving money, that's a serious bug.

**What it should be:** an order line should save the unit price at the moment it's added, and a paid order should be locked to the amount that was paid.

---

## 5. Mixing up IDs and indexes in the functions

Some functions take an **ID**, some take an **index**, and both are plain `int`:

void   printOrder(int orderId);          // takes an ID
void   markOrderPaid(int orderId);       // takes an ID
double calculateOrderTotal(int orderIndex); // takes an INDEX
int    createOrder(...);                 // RETURNS an index (and nobody uses it)

**What can go wrong:** nothing stops someone from calling `calculateOrderTotal(1001)` with an order ID. There's no bounds check, so it reads way past the end of the arrays, which is undefined behavior in C++. It might return garbage or it might crash. The index is an internal storage detail and should never have been visible outside.

---

## 6. Errors are just printed, and the caller never finds out

Almost every function handles errors like this:

cout << "ERROR: customer id " << customerId << " not found.\n";
return;

and most of them return `void`.

**What can go wrong:** the code that called the function has no way of knowing it failed. In `runDemoScenario`, if `createOrder(1001, ...)` fails, the next lines still try to add products to order 1001 as if it worked. The program fails silently instead of failing loudly.

---

## 7. Business logic and screen output are mixed together

The functions that apply the real rules (stock, validation, totals) are also the ones calling `cout`. `printOrder`, for example, does three jobs at once: it looks up the order, calculates the total, and prints it.

**What can go wrong:**
- You can't reuse the logic with a different interface (web API, GUI, tests) without it printing to the console.
- `cout << fixed << setprecision(2)` inside these functions changes the formatting of the console stream for the whole program as a side effect.

---

## 8. Business rules are scattered instead of living with the data they protect

- The VIP discount is a hardcoded `0.90` buried inside `calculateOrderTotal`. If the discount changes, or a second discount type is added, you have to go hunt for it.
- The rule "a paid order can't be changed" only exists inside `addLineToOrder`. Any new function that touches an order has to *remember* to check it, and nothing forces it to.

**What it should be:** the `Order` itself should refuse to be changed after it's paid. Then nobody has to remember, because it's impossible to do it wrong.

---

## 9. Missing validation

- `addProduct` accepts a negative price or negative stock.
- `addCustomer` accepts an empty name and an empty or invalid email.
- `createOrder` takes the date as a free `string`, so `"banana"` is a valid date.
- `markOrderPaid` doesn't check whether the order is already paid, so you can pay twice without any message. There's no real idea of an order "state".

---

## 10. Stock is reserved forever

`addLineToOrder` takes the quantity out of stock **the moment the line is added**, before the order is paid. And there's no way to cancel an order or remove a line.

**What can go wrong:** every unpaid order holds onto its stock forever. If a customer creates an order and never pays, those items are gone from stock permanently.

---

## 11. The same product can be added as multiple lines

If you add product 101 twice to the same order, you get two separate lines instead of one line with a bigger quantity. It looks messy, and it also uses up the 20-line limit for no reason.

---

## 12. `double` is used for money


double productPrices[MAX_PRODUCTS];


Floating point numbers can't represent values like `0.1` exactly, so totals and discounts can end up off by small amounts. For money you want an exact decimal type (`decimal` in C#).

---

## 13. Fixed hard limits

const int MAX_CUSTOMERS = 50;
const int MAX_ORDERS = 100;
const int MAX_LINES_PER_ORDER = 20;


Customer number 51 just can't be added. On top of that, the 2D line arrays reserve `100 × 20 = 2000` slots each from the start, even if the program only ever has 3 orders.

---

## 14. Smaller things worth mentioning

- **Magic `-1`:** the find functions return `-1` for "not found", and every caller has to remember to check for it.
- **Copy-paste code:** `findCustomerIndexById`, `findProductIndexById` and `findOrderIndexById` are the same loop three times, and the "order not found" check is repeated in several functions.
- **Menu is missing features:** there are `addCustomer` and `addProduct` functions, but the menu has no option for them. Customers and products only come from the seed data.
- **Fragile input:** if you type letters instead of a number in the menu, `cin` fails, `choice` becomes `0`, and the program just exits with "Bye.".
- `cin >> date` stops at the first space.
- Strings are passed by value, so they get copied on every call.
- `using namespace std;` at global scope.
- Every lookup is a linear search through the array.

---

## The root problem

All of these come back to one thing: **the data and the behavior are separated, and nothing owns anything.** The data sits in global arrays, the rules are spread across free functions, and every rule depends on the programmer remembering it.

In the C# version, every piece of data should have an owner that protects it:

- `Customer` holds its own data together in one object.
- `Product` owns its stock and is the only one allowed to reduce it.
- `OrderLine` points to a real `Product` and saves the unit price when it's created.
- `Order` points to a real `Customer`, owns its lines, refuses changes after it's paid, and calculates its own total.
- A store or system object owns the collections instead of global variables, and the console menu lives in its own class, separate from the business logic.
