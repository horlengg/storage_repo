<br>
<br>

# 10 Javascript array method usefull for development

<br>
<br>

"10 JavaScript Array Methods Useful for Development" is your go-to guide for mastering essential array operations in modern JavaScript. Whether you're a beginner or experienced developer, this post breaks down the top 10 most powerful and frequently used array methods—including reduce(), map(),flatMap(), filter(),every(), find(), some(), shift(),unshift(), and sort().

Each method is explained with simple syntax, clear examples, and real-world use cases to help you write cleaner, more efficient, and more readable code. If you're building anything from simple web apps to complex front-end projects, knowing these methods is a must for faster development and better performance.


<br>

## Contents
1. [Array.prototype.reduce()](#arrayprototypereduce)
2. [Array.prototype.map()](#arrayprototypemap)
3. [Array.prototype.flatMap()](#arrayprototypeflatmap)
4. [Array.prototype.filter()](#arrayprototypefilter)
5. [Array.prototype.every()](#arrayprototypeevery)
6. [Array.prototype.find()](#arrayprototypefind)
7. [Array.prototype.some()](#arrayprototypesome)
8. [Array.prototype.shift()](#arrayprototypeshift)
9. [Array.prototype.unshift()](#arrayprototypeunshift)
10. [Array.prototype.sort()](#arrayprototypesort)


<br>

## Array.prototype.reduce()

The reduce() method of Array instances executes a user-supplied "reducer" callback function on each element of the array, in order, passing in the return value from the calculation on the preceding element. The final result of running the reducer across all elements of the array is a single value.

The first time that the callback is run there is no "return value of the previous calculation". If supplied, an initial value may be used in its place. Otherwise the array element at index 0 is used as the initial value and iteration starts from the next element (index 1 instead of index 0).


```javascript
const productList = [
    {
        id : 1,
        title: "Milk",
        price : 1.25
    },
    {
        id : 2,
        title: "Beer",
        price : 2.5
    },
    {
        id : 3,
        title: "Food",
        price : 1.0
    },
];

const totalAmount = productList.reduce((acc, curr)=> acc + curr.price ,0) // 0 is default value of acc

console.log(`Total amount ::: \$${totalAmount}`); //Total amount ::: $4.75

```

<br>
<br>
<br>

## Array.prototype.map()
The map() method of Array instances creates a new array populated with the results of calling a provided function on every element in the calling array.


```javascript

const users = [
    {
        id : 1,
        name: "Sok",
        role : "ADMIN"
    },
    {
        id : 2,
        name: "Rith",
        role : "STAFF"
    },
    {
        id : 3,
        name: "Leng",
        role : "ADMIN"
    },
];


const listAllUserIds = users.map(e => e.id);

console.log(listAllUserIds); // [1,2,3]
```

<br>
<br>
<br>

## Array.prototype.flatMap()
The flatMap() method of Array instances returns a new array formed by applying a given callback function to each element of the array, and then flattening the result by one level. It is identical to a map() followed by a flat() of depth 1 (arr.map(...args).flat()), but slightly more efficient than calling those two methods separately.


```javascript
const orderList = [
    {
        orderNo : '00000000001',
        user : 'Makara',
        date : '2025-07-05T13:20:03.275Z',
        products : [
            {
                id : 1,
                name : 'Coca-Cola',
                price : 1.2,
                qty : 1
            },
            {
                id : 101,
                name : 'Pepsi',
                price : 0.5,
                qty : 2
            },
        ]
    },
    {
        orderNo : '00000000002',
        user : 'Seyha',
        date : '2025-06-05T13:20:03.275Z',
        products : [
            {
                id : 1,
                name : 'Coca-Cola',
                price : 1.2,
                qty : 10
            },
            {
                id : 101,
                name : 'Pepsi',
                price : 0.5,
                qty : 3
            },
             {
                id : 20,
                name : 'Carabao',
                price : 1.1,
                qty : 8
            },
        ]
    },
]

const listProductOrders = orderList.flatMap(orderItem => orderItem.products);
console.log(listProductOrders)
/**
 Output :
[
  { id: 1, name: 'Coca-Cola', price: 1.2, qty: 1 },
  { id: 101, name: 'Pepsi', price: 0.5, qty: 2 },
  { id: 1, name: 'Coca-Cola', price: 1.2, qty: 10 },
  { id: 101, name: 'Pepsi', price: 0.5, qty: 3 },
  { id: 20, name: 'Carabao', price: 1.1, qty: 8 }
]

 */

```


<br>
<br>
<br>

## Array.prototype.filter()
The filter() method of Array instances creates a shallow copy of a portion of a given array, filtered down to just the elements from the given array that pass the test implemented by the provided function.


```javascript

const users = [
    {
        id : 1,
        name: "Sok",
        role : "ADMIN"
    },
    {
        id : 2,
        name: "Rith",
        role : "STAFF"
    },
    {
        id : 3,
        name: "Leng",
        role : "ADMIN"
    },
];


const listAllAdminUsers = users.filter(e => e.role == 'ADMIN');

console.log(listAllAdminUsers); 
/*

ouput :
[
  { id: 1, name: 'Sok', role: 'ADMIN' },
  { id: 3, name: 'Leng', role: 'ADMIN' }
]

*/
```


<br>
<br>
<br>

## Array.prototype.every()
The every() method of Array instances tests whether all elements in the array pass the test implemented by the provided function. It returns a Boolean value.


```javascript
const cart = [
  { name: "Laptop", inStock: true },
  { name: "Mouse", inStock: true },
  { name: "Keyboard", inStock: true },
];

const allAvailable = cart.every(item => item.inStock);
console.log(allAvailable); // true
```

<br>
<br>
<br>

## Array.prototype.find()
The find() method of Array instances returns the first element in the provided array that satisfies the provided testing function. If no values satisfy the testing function, undefined is returned.

```javascript
const users = [
  { id: 1, name: "Leng" },
  { id: 2, name: "Dev" },
  { id: 3, name: "Thou" },
];

const user = users.find(u => u.name === "Leng");
console.log(user); // { id: 1, name: "Leng" }

```

<br>
<br>
<br>


## Array.prototype.some()
The some() method of Array instances tests whether at least one element in the array passes the test implemented by the provided function. It returns true if, in the array, it finds an element for which the provided function returns true; otherwise it returns false. It doesn't modify the array.

```javascript

const items = [
  { name: "Pen", stock: 10 },
  { name: "Notebook", stock: 0 },
  { name: "Eraser", stock: 5 },
];

const hasOutOfStock = items.some(item => item.stock === 0);
console.log(hasOutOfStock); // true


```

<br>
<br>
<br>


## Array.prototype.shift()
The shift() method of Array instances removes the first element from an array and returns that removed element. This method changes the length of the array.

```javascript

const queue = ["first", "second", "third"];

const removed = queue.shift();

console.log(removed); // "first"
console.log(queue);   // ["second", "third"]

```

<br>
<br>
<br>


## Array.prototype.unshift()
The unshift() method of Array instances adds the specified elements to the beginning of an array and returns the new length of the array.


```javascript
const fruits = ["banana", "orange"];

const newLength = fruits.unshift("apple");

console.log(fruits);     // ["apple", "banana", "orange"]
console.log(newLength);  // 3

```


<br>
<br>
<br>

## Array.prototype.sort()
The sort() method of Array instances sorts the elements of an array in place and returns the reference to the same array, now sorted. The default sort order is ascending, built upon converting the elements into strings, then comparing their sequences of UTF-16 code unit values.

The time and space complexity of the sort cannot be guaranteed as it depends on the implementation.

To sort the elements in an array without mutating the original array, use toSorted().


```javascript
const studentList = [
    {
        name : 'Leng',
        score : 50
    },
    {
        name : 'Ravuth',
        score : 70
    },
    {
        name : 'Pisey',
        score : 30
    },
    {
        name : 'Roth',
        score : 60
    },
]

// sort by score :: ascending
studentList.sort((a,b)=> a.score - b.score )
console.log(studentList)

// sort by score :: descending
studentList.sort((a,b)=> b.score - a.score )
console.log(studentList)

//sort by name :: ascending
studentList.sort((a,b)=> {
    const compareName = a.name.toUpperCase() > b.name.toUpperCase();
    return compareName ? 1 : -1;
} );
console.log(studentList)

//sort by name :: descending
studentList.sort((a,b)=> {
    const compareName = a.name.toUpperCase() > b.name.toUpperCase();
    return compareName ? -1 : 1;
} );
console.log(studentList)


/*

ouput :
[
  { name: 'Pisey', score: 30 },
  { name: 'Leng', score: 50 },
  { name: 'Roth', score: 60 },
  { name: 'Ravuth', score: 70 }
]
[
  { name: 'Ravuth', score: 70 },
  { name: 'Roth', score: 60 },
  { name: 'Leng', score: 50 },
  { name: 'Pisey', score: 30 }
]
[
  { name: 'Leng', score: 50 },
  { name: 'Pisey', score: 30 },
  { name: 'Ravuth', score: 70 },
  { name: 'Roth', score: 60 }
]
[
  { name: 'Roth', score: 60 },
  { name: 'Ravuth', score: 70 },
  { name: 'Pisey', score: 30 },
  { name: 'Leng', score: 50 }
]

*/
```

<br>
<br>
<br>

Thank you guy!. ):
