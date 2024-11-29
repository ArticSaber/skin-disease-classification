Here's the solution with separate HTML and JavaScript files for the All Products and Add Product pages:

```
---

Project Structure

/project-directory
    /css
        bootstrap.min.css (Bootstrap CSS from CDN)
    /js
        allproducts.js (JavaScript for All Products page)
        addproduct.js (JavaScript for Add Product page)
    allproducts.html
    addproduct.html


---

allproducts.html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>All Products</title>
    <link rel="stylesheet" href="css/bootstrap.min.css">
</head>
<body>
    <div class="container mt-5">
        <h1 class="text-center">All Products</h1>
        <table class="table table-bordered mt-4">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Image</th>
                    <th>Product Name</th>
                    <th>Product Price</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody id="productTable"></tbody>
        </table>
        <a href="addproduct.html" class="btn btn-primary">Add Product</a>
    </div>

    <script src="js/allproducts.js"></script>
</body>
</html>


---

js/allproducts.js

// Array to store product data
let products = [
    { id: 1, image: "https://via.placeholder.com/50", name: "Product 1", price: "$100" },
    { id: 2, image: "https://via.placeholder.com/50", name: "Product 2", price: "$150" }
];

// Load products into the table
function loadProducts() {
    const productTable = document.getElementById("productTable");
    productTable.innerHTML = ""; // Clear previous entries

    products.forEach((product) => {
        const row = document.createElement("tr");

        row.innerHTML = `
            <td>${product.id}</td>
            <td><img src="${product.image}" alt="Product Image" class="img-thumbnail" width="50"></td>
            <td contenteditable="true">${product.name}</td>
            <td contenteditable="true">${product.price}</td>
            <td>
                <button class="btn btn-danger btn-sm" onclick="deleteProduct(${product.id})">Delete</button>
            </td>
        `;
        productTable.appendChild(row);
    });
}

// Delete a product
function deleteProduct(id) {
    products = products.filter((product) => product.id !== id);
    loadProducts();
}

// Load products on page load
document.addEventListener("DOMContentLoaded", loadProducts);


---

addproduct.html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Add Product</title>
    <link rel="stylesheet" href="css/bootstrap.min.css">
</head>
<body>
    <div class="container mt-5">
        <h1 class="text-center">Add Product</h1>
        <form id="addProductForm" class="mt-4">
            <div class="mb-3">
                <label for="productId" class="form-label">Product ID</label>
                <input type="number" class="form-control" id="productId" required>
            </div>
            <div class="mb-3">
                <label for="productImage" class="form-label">Product Image URL</label>
                <input type="text" class="form-control" id="productImage" required>
            </div>
            <div class="mb-3">
                <label for="productName" class="form-label">Product Name</label>
                <input type="text" class="form-control" id="productName" required>
            </div>
            <div class="mb-3">
                <label for="productPrice" class="form-label">Product Price</label>
                <input type="text" class="form-control" id="productPrice" required>
            </div>
            <button type="submit" class="btn btn-primary">Add Product</button>
        </form>
    </div>

    <script src="js/addproduct.js"></script>
</body>
</html>


---

js/addproduct.js

// Handle the form submission
document.getElementById("addProductForm").addEventListener("submit", function (event) {
    event.preventDefault();

    // Get product details from form inputs
    const product = {
        id: parseInt(document.getElementById("productId").value),
        image: document.getElementById("productImage").value,
        name: document.getElementById("productName").value,
        price: document.getElementById("productPrice").value
    };

    // Save product to localStorage
    const existingProducts = JSON.parse(localStorage.getItem("products")) || [];
    existingProducts.push(product);
    localStorage.setItem("products", JSON.stringify(existingProducts));

    // Redirect to the All Products page
    window.location.href = "allproducts.html";
});


---

Modifications for js/allproducts.js (to use localStorage)

Update the allproducts.js file to read from localStorage:

// Load products into the table
function loadProducts() {
    const productTable = document.getElementById("productTable");

    // Fetch products from localStorage
    products = JSON.parse(localStorage.getItem("products")) || [
        { id: 1, image: "https://via.placeholder.com/50", name: "Product 1", price: "$100" },
        { id: 2, image: "https://via.placeholder.com/50", name: "Product 2", price: "$150" }
    ];

    productTable.innerHTML = ""; // Clear previous entries

    products.forEach((product) => {
        const row = document.createElement("tr");

        row.innerHTML = `
            <td>${product.id}</td>
            <td><img src="${product.image}" alt="Product Image" class="img-thumbnail" width="50"></td>
            <td contenteditable="true">${product.name}</td>
            <td contenteditable="true">${product.price}</td>
            <td>
                <button class="btn btn-danger btn-sm" onclick="deleteProduct(${product.id})">Delete</button>
            </td>
        `;
        productTable.appendChild(row);
    });
}

// Delete a product
function deleteProduct(id) {
    products = products.filter((product) => product.id !== id);
    localStorage.setItem("products", JSON.stringify(products));
    loadProducts();
}

// Load products on page load
document.addEventListener("DOMContentLoaded", loadProducts);


---

How It Works:

1. allproducts.html:

Displays a table of products.

Reads from localStorage to populate products.

Provides the ability to delete products.



2. addproduct.html:

Provides a form to add a new product.

Saves the new product to localStorage and redirects to allproducts.html.



3. LocalStorage:

Used to persist product data across pages.



4. Contenteditable:

Allows editing product names and prices directly in the table (no need for separate edit forms).




Place the files in the specified structure, open allproducts.html in a browser, and test the functionality!

```
