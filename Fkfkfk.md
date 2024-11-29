Here's an updated version of the JavaScript and necessary additions for the "View" and "Edit" functionalities. The following changes have been made:

Key Changes

1. View Button: Opens a modal to show product details (ID, Image, Name, and Cost).


2. Edit Button: Makes productName and productCost fields editable. Replaces the "Edit" button with "Save" and "Cancel".


3. Save Button: Saves the updated product details to localStorage and refreshes the data.


4. Cancel Button: Restores the previous state without saving changes.




---

Updated Code

HTML for Modal (Add in product-list-array-demo.html)

<!-- Modal -->
<div class="modal fade" id="productModal" tabindex="-1" aria-labelledby="productModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title" id="productModalLabel">Product Details</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>
      <div class="modal-body">
        <p><strong>ID:</strong> <span id="modalProductId"></span></p>
        <p><strong>Name:</strong> <span id="modalProductName"></span></p>
        <p><strong>Cost:</strong> <span id="modalProductCost"></span></p>
        <p>
          <strong>Image:</strong><br />
          <img id="modalProductImage" src="" width="200" height="120" />
        </p>
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
      </div>
    </div>
  </div>
</div>

Updated JavaScript in product.js

function loadData() {
  let allProductsString = localStorage.getItem("allProducts");
  if (allProductsString != null) {
    allProducts = JSON.parse(allProductsString);
    let content = "";
    for (let eachProduct of allProducts) {
      content += `<tr>
                      <td>${eachProduct.id}</td>
                      <td><img src="${eachProduct.productImageUrl}" width="80" height="50"></td>
                      <td contenteditable="false" id="name-${eachProduct.id}">${eachProduct.productName}</td>
                      <td contenteditable="false" id="cost-${eachProduct.id}">${eachProduct.productCost}</td>
                      <td><button type="button" class="btn btn-warning" onclick="viewProduct(${eachProduct.id})">View</button></td>
                      <td><button type="button" class="btn btn-primary" onclick="editProduct(${eachProduct.id})" id="edit-${eachProduct.id}">Edit</button></td>
                      <td><button type="button" class="btn btn-danger" onclick="deleteProduct(${eachProduct.id})">Delete</button></td>
                  </tr>`;
    }
    document.getElementById("data").innerHTML = content;
  } else {
    document.getElementById("message").innerHTML = "No Products to display!";
  }
}

function viewProduct(id) {
  let product = allProducts.find((eachProduct) => eachProduct.id == id);

  // Populate modal with product data
  document.getElementById("modalProductId").innerText = product.id;
  document.getElementById("modalProductName").innerText = product.productName;
  document.getElementById("modalProductCost").innerText = product.productCost;
  document.getElementById("modalProductImage").src = product.productImageUrl;

  // Show the modal
  let productModal = new bootstrap.Modal(document.getElementById("productModal"));
  productModal.show();
}

function editProduct(id) {
  // Make fields editable
  document.getElementById(`name-${id}`).contentEditable = "true";
  document.getElementById(`cost-${id}`).contentEditable = "true";

  // Change "Edit" button to "Save" and "Cancel"
  let editButton = document.getElementById(`edit-${id}`);
  editButton.outerHTML = `
    <button type="button" class="btn btn-success" onclick="saveProduct(${id})" id="save-${id}">Save</button>
    <button type="button" class="btn btn-secondary" onclick="cancelEdit(${id})" id="cancel-${id}">Cancel</button>
  `;
}

function saveProduct(id) {
  // Save updated values
  let updatedName = document.getElementById(`name-${id}`).innerText;
  let updatedCost = document.getElementById(`cost-${id}`).innerText;

  // Update product in the array
  let productIndex = allProducts.findIndex((eachProduct) => eachProduct.id == id);
  allProducts[productIndex].productName = updatedName;
  allProducts[productIndex].productCost = updatedCost;

  // Save to localStorage
  localStorage.setItem("allProducts", JSON.stringify(allProducts));

  // Reload data
  loadData();
}

function cancelEdit(id) {
  // Reload data without saving
  loadData();
}


---

How It Works

1. View Product:

Clicking "View" opens a modal displaying product details.



2. Edit Product:

Clicking "Edit" makes the Name and Cost fields editable.

Displays "Save" and "Cancel" buttons.



3. Save Changes:

Updates the product in the array and persists it to localStorage.



4. Cancel Edit:

Reloads the table without saving any changes.




Let me know if you need further assistance!

