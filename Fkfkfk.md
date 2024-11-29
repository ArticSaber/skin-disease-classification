Below is the full code with separate HTML files for the "Add Product" page and the "Product Table" page, along with a shared JavaScript file. When a product is added, the user is redirected to the "Products" page.


---

Directory Structure

project/
│
├── add-product.html
├── product-table.html
├── styles.css (optional)
└── scripts.js


---

1. Add Product Page (add-product.html)

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Add Product</title>
  <link 
    rel="stylesheet" 
    href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css"
  >
</head>
<body>
  <div class="container mt-5">
    <h2 class="mb-4">Add Product</h2>

    <!-- Add Product Form -->
    <form id="addProductForm">
      <div class="form-row">
        <div class="form-group col-md-4">
          <input type="text" class="form-control" id="productName" placeholder="Product Name" required>
        </div>
        <div class="form-group col-md-4">
          <input type="url" class="form-control" id="productUrl" placeholder="Product URL" required>
        </div>
        <div class="form-group col-md-2">
          <input type="number" class="form-control" id="productPrice" placeholder="Product Price" required>
        </div>
        <button type="submit" class="btn btn-primary">Add Product</button>
      </div>
    </form>
  </div>

  <script src="scripts.js"></script>
</body>
</html>


---

2. Product Table Page (product-table.html)

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Product Table</title>
  <link 
    rel="stylesheet" 
    href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css"
  >
</head>
<body>
  <div class="container mt-5">
    <h2 class="mb-4">Product Table</h2>
    <a href="add-product.html" class="btn btn-success mb-3">Add Product</a>

    <!-- Product Table -->
    <table class="table table-bordered" id="productTable">
      <thead>
        <tr>
          <th>ID</th>
          <th>Name</th>
          <th>URL</th>
          <th>Price</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        <!-- Dynamic Rows -->
      </tbody>
    </table>
  </div>

  <script src="scripts.js"></script>
</body>
</html>


---

3. JavaScript File (scripts.js)

// Simulating a simple local database
const productKey = "productList";

// Redirect from Add Product Form to Product Table
if (document.getElementById("addProductForm")) {
  document.getElementById("addProductForm").addEventListener("submit", (e) => {
    e.preventDefault();

    const name = document.getElementById("productName").value;
    const url = document.getElementById("productUrl").value;
    const price = document.getElementById("productPrice").value;

    const products = JSON.parse(localStorage.getItem(productKey)) || [];
    const newProduct = {
      id: products.length ? products[products.length - 1].id + 1 : 1,
      name,
      url,
      price,
    };

    products.push(newProduct);
    localStorage.setItem(productKey, JSON.stringify(products));

    window.location.href = "product-table.html"; // Redirect to Product Table
  });
}

// Render Product Table
if (document.querySelector("#productTable")) {
  const products = JSON.parse(localStorage.getItem(productKey)) || [];
  renderProducts(products);

  function renderProducts(productList) {
    const tableBody = document.querySelector("#productTable tbody");
    tableBody.innerHTML = "";

    productList.forEach((product) => {
      const row = document.createElement("tr");

      row.innerHTML = `
        <td>${product.id}</td>
        <td contenteditable="false">${product.name}</td>
        <td contenteditable="false">${product.url}</td>
        <td contenteditable="false">${product.price}</td>
        <td>
          <button class="btn btn-sm btn-warning edit-btn">Edit</button>
          <button class="btn btn-sm btn-success save-btn d-none">Save</button>
          <button class="btn btn-sm btn-secondary cancel-btn d-none">Cancel</button>
          <button class="btn btn-sm btn-info view-btn">View</button>
          <button class="btn btn-sm btn-danger delete-btn">Delete</button>
        </td>
      `;

      // Action Button Event Listeners
      row.querySelector(".edit-btn").addEventListener("click", () => toggleEdit(row, true));
      row.querySelector(".save-btn").addEventListener("click", () => saveEdit(row, product.id));
      row.querySelector(".cancel-btn").addEventListener("click", () => toggleEdit(row, false));
      row.querySelector(".view-btn").addEventListener("click", () => viewProduct(product));
      row.querySelector(".delete-btn").addEventListener("click", () => deleteProduct(product.id, row));

      tableBody.appendChild(row);
    });
  }

  // Toggle Edit Mode
  function toggleEdit(row, isEditing) {
    const cells = row.querySelectorAll("td[contenteditable]");
    const editBtn = row.querySelector(".edit-btn");
    const saveBtn = row.querySelector(".save-btn");
    const cancelBtn = row.querySelector(".cancel-btn");

    cells.forEach((cell) => (cell.contentEditable = isEditing));
    editBtn.classList.toggle("d-none", isEditing);
    saveBtn.classList.toggle("d-none", !isEditing);
    cancelBtn.classList.toggle("d-none", !isEditing);
  }

  // Save Edited Product
  function saveEdit(row, productId) {
    const products = JSON.parse(localStorage.getItem(productKey));
    const product = products.find((p) => p.id === productId);
    const cells = row.querySelectorAll("td[contenteditable]");
    product.name = cells[0].textContent.trim();
    product.url = cells[1].textContent.trim();
    product.price = cells[2].textContent.trim();
    localStorage.setItem(productKey, JSON.stringify(products));
    toggleEdit(row, false);
  }

  // Delete Product
  function deleteProduct(productId, row) {
    const products = JSON.parse(localStorage.getItem(productKey));
    const updatedProducts = products.filter((product) => product.id !== productId);
    localStorage.setItem(productKey, JSON.stringify(updatedProducts));
    row.remove();
  }

  // View Product Details
  function viewProduct(product) {
    const details = `
      Product Details:
      ID: ${product.id}
      Name: ${product.name}
      URL: ${product.url}
      Price: ${product.price}
    `;
    alert(details);
  }
}


---

How It Works

1. Add Product Page:

Enter product details.

On form submission, the product is added to localStorage and redirects to the Product Table page.



2. Product Table Page:

Loads the product list from localStorage.

Enables edit, save, cancel, view, and delete functionalities for each product.



3. LocalStorage:

Products are stored in the browser’s localStorage under the key productList.




This setup ensures a clean separation between pages and functionality while leveraging localStorage for simple data persistence.

