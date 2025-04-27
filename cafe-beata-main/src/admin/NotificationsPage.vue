<template>
  <div class="notifications-page">
    <!-- Sidebar Toggle Button (For Mobile) -->
    <button class="menu-button" @click="toggleSidebar">
      <div class="menu-icon-container">
        <i class="fa fa-bars"></i>
      </div>
    </button>

    <div v-if="isSidebarOpen" class="overlay"></div>
    
    <!-- Sidebar -->
    <div :class="['sidebar', { 'open': isSidebarOpen }]" @click.stop>
      <button class="close-sidebar" @click="toggleSidebar">
        <i class="fa fa-times"></i>
      </button>
      
      <!-- Admin Profile Section -->
      <div class="user-profile-section">
        <h3><i class="fa fa-user-circle"></i>Cafe Staff Dashboard</h3>
      </div>

      <hr class="utility-divider">

      <!-- Utility Buttons -->
      <div class="utility-section">
        <button @click="goToOrderRecord" class="utility-button">
          <i class="fa fa-history"></i>
          <span>View Order Record</span>
        </button>

        <button @click="toggleMenuEditor" class="utility-button">
          <i class="fa fa-utensils"></i>
          <span>Menu Editor</span>
        </button>

        <button @click="toggleStockManager" class="utility-button">
          <i class="fa fa-boxes"></i>
          <span>Stock Management</span>
        </button>

        <button @click="toggleChangePassword" class="utility-button">
          <i class="fa fa-key"></i>
          <span>Change Password</span>
        </button>

        <button @click="logout" class="utility-button logout">
          <i class="fa fa-sign-out"></i>
          <span>Logout</span>
        </button>
      </div>

      <!-- Cafe Status Section -->
      <div class="cafe-status-section">
        <button @click="toggleCafeStatus" :class="{'open-btn': isCafeOpen, 'closed-btn': !isCafeOpen}" class="cafe-toggle-btn">
          <i :class="isCafeOpen ? 'fa fa-check-circle' : 'fa fa-times-circle'"></i>
          {{ isCafeOpen ? 'Set Cafe Closed' : 'Set Cafe Open' }}
        </button>
      </div>
    </div>

    <!-- Main Content -->
    <div :class="['content', { 'shifted': isSidebarOpen }]">
      <!-- Add the top bar with pink gradient at the very top -->
      <div class="top-bar">
        <div class="centered-content">
          <!-- Add sidebar button to the top bar -->
          <button class="menu-button-header" @click="toggleSidebar">
            <div class="menu-icon-container">
              <i class="fa fa-bars"></i>
            </div>
          </button>
          <div class="logo-container">
            <div class="cafe-title">Cafe Preorder Cafe Staff Dashboard</div>
          </div>
        </div>
      </div>

      <div class="content-below-top-bar">
        <div v-if="notificationVisible" :class="['notification-popup', notificationClass]">
          <p>{{ notificationMessage }}</p>
        </div>

        <!-- Search Bar -->
        <div class="search-bar">
          <input 
            type="text" 
            v-model="searchQuery" 
            placeholder="Search orders by ID, customer name..." 
            class="search-input"
          />
        </div>

        <div v-if="isLoading" class="loading">Loading...</div>

        <div v-if="filteredOrders.length && !isLoading" class="orders-container">
          <h2>Pending Orders</h2>
          <div class="orders-list">
            <div class="order-item" 
              v-for="order in filteredOrders" 
              :key="order.id" 
              :data-order-id="order.id"
              :class="{
                'order-declined-state': activeDeclineOrderId === order.id,
                'order-pending-approval': order.isPendingApproval
              }">
              <div class="order-details">
                <h3>Order ID: {{ order.id }}</h3>
                <p><strong>Customer:</strong> {{ order.customer_name }}</p>
                <p><strong>Status:</strong> {{ order.status }}{{ order.isPendingApproval ? ' (Pending Approval)' : '' }}</p>
                <p><strong>Time Order: </strong> {{ timeAgo(order.created_at) }}</p> <!-- Time Ago Display -->

                <div class="items-section">
                  <strong>Items:</strong>
                  <ul>
                    <li v-for="item in order.items" :key="item.name">
                      {{ item.name }} - ₱{{ item.price }} x {{ item.quantity }}
                    </li>
                  </ul>
                </div>

                <!-- Total Amount below items -->
                <div class="order-total">
                  <p><strong>Total Amount: ₱{{ calculateOrderTotal(order.items) }}</strong></p>
                </div>
              </div>

              <div class="order-actions">
                <!-- Mark as Completed Buttons -->
                <button 
                  @click="showCompletionConfirmation(order.id)" 
                  class="mark-completed-btn small-btn"
                  :disabled="!orderReadyStatus[order.id]"
                  :class="{ 'disabled': !orderReadyStatus[order.id] }"
                >
                  Mark as Completed
                </button>

                <!-- Order Ready button -->
                <button 
                  @click="sendOrderReadyNotification(order.id, order.customer_name, order.items)" 
                  class="order-ready-btn small-btn"
                >
                  Order Ready  🔔
                </button>

                <!-- Decline button -->
                <button @click="openDeclineDialog(order)" class="decline-btn">
                  Decline
                </button>
              </div>
            </div>
          </div>
          
          <!-- Completion Confirmation Popup -->
          <div v-if="confirmCompleteOrderId" class="completion-confirmation-popup">
            <div class="completion-confirmation-content">
              <h3>Confirm Completion</h3>
              <p>Are you sure Order ID: {{ confirmCompleteOrderId }} is completed?</p>
              <div class="confirmation-buttons">
                <button @click="confirmCompletion" class="confirm-yes-btn">Yes</button>
                <button @click="cancelCompletion" class="confirm-no-btn">No</button>
              </div>
            </div>
          </div>
        </div>

        <div v-else-if="!isLoading" class="no-orders">
          <p>No pending orders at the moment.</p>
        </div>

        <!-- Popup Notification Sent -->
        <div v-if="notificationSent" class="notification-sent-popup">
          <p>Notification Sent!</p>
          <button @click="notificationSent = false" class="close-popup-btn">Close</button>
        </div>

        <!-- Menu Editor Popup Modal -->
        <div v-if="showMenuEditor" class="menu-editor-modal">
          <div class="menu-editor-content">
            <div class="menu-editor-header">
              <h2>Menu Editor</h2>
              <button @click="toggleMenuEditor" class="close-modal-btn">
                <i class="fa-solid fa-times"></i>
              </button>
            </div>
            <div class="menu-editor-body">
              <ItemEditor />
            </div>
          </div>
        </div>

        <!-- Stock Management Modal -->
        <div v-if="showStockManager" class="stock-manager-modal">
          <div class="stock-manager-content">
            <div class="stock-manager-header">
              <h2>Stock Management</h2>
              <button @click="toggleStockManager" class="close-modal-btn">
                <i class="fa-solid fa-times"></i>
              </button>
            </div>
            <div class="stock-manager-body">
              <!-- Search Bar -->
              <div class="stock-search-bar">
                <div class="stock-filters">
                  <input 
                    type="text" 
                    v-model="stockSearchQuery" 
                    placeholder="Search items..." 
                    class="search-input"
                  />
                  <select v-model="selectedCategory" class="category-filter">
                    <option value="">All Categories</option>
                    <option v-for="category in uniqueCategories" :key="category" :value="category">
                      {{ category }}
                    </option>
                  </select>
                </div>
              </div>

              <!-- Stock Items Table -->
              <div class="stock-table-container">
                <table class="stock-table">
                  <thead>
                    <tr>
                      <th>Item Name</th>
                      <th>Category</th>
                      <th>Current Stock</th>
                      <th>Status</th>
                      <th>Actions</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="item in filteredStockItems" :key="item.id">
                      <td>{{ item.name }}</td>
                      <td>{{ item.category }}</td>
                      <td>{{ item.quantity >= 999999 ? 'Unlimited' : item.quantity }}</td>
                      <td :class="getStockStatusClass(item)">{{ getStockStatus(item) }}</td>
                      <td>
                        <button @click="openStockUpdateModal(item)" class="update-stock-btn">
                          Update Stock
                        </button>
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        </div>

        <!-- Stock Update Modal -->
        <div v-if="showStockUpdateModal" class="stock-update-modal">
          <div class="stock-update-content">
            <h3>Update Stock: {{ selectedItem?.name }}</h3>
            <div class="stock-update-form">
              <div class="form-group">
                <label>Current Stock: {{ selectedItem?.quantity >= 999999 ? 'Unlimited' : selectedItem?.quantity }}</label>
              </div>
              <div class="form-group">
                <label>Action:</label>
                <select v-model="stockUpdateAction">
                  <option value="add">Add Stock</option>
                  <option value="subtract">Remove Stock</option>
                  <option value="set">Set Stock</option>
                  <option value="disabled">Disabled (Out of Stock)</option>
                  <option value="enabled">Enabled (Unlimited Orders)</option>
                </select>
              </div>
              <div class="form-group">
                <label>Quantity:</label>
                <input 
                  v-if="stockUpdateAction !== 'disabled' && stockUpdateAction !== 'enabled'"
                  type="number" 
                  v-model.number="stockUpdateQuantity" 
                  min="0"
                  :max="stockUpdateAction === 'subtract' ? selectedItem?.quantity : 999999"
                />
                <span v-else-if="stockUpdateAction === 'disabled'">
                  Item will be marked as out of stock and cannot be ordered.
                </span>
                <span v-else-if="stockUpdateAction === 'enabled'">
                  Item will be available for unlimited orders regardless of quantity.
                </span>
              </div>
              <div class="form-group">
                <label>Reason:</label>
                <input type="text" v-model="stockUpdateReason" placeholder="Enter reason for update"/>
              </div>
              <div class="update-buttons">
                <button @click="submitStockUpdate" class="confirm-btn">Update Stock</button>
                <button @click="closeStockUpdateModal" class="cancel-btn">Cancel</button>
              </div>
            </div>
          </div>
        </div>

        <!-- Decline Modal -->
        <div v-if="showDeclineModal" class="decline-modal-overlay">
          <div class="decline-modal-content">
            <div class="decline-modal-header">
              <h3>Order Adjustment</h3>
              <button @click="closeDeclineModal" class="close-modal-btn">✕</button>
      </div>
            <div class="decline-modal-body">
              <p><strong>Order ID:</strong> {{ selectedOrder?.id }}</p>
              <p><strong>Customer:</strong> {{ selectedOrder?.customer_name }}</p>
              
              <!-- Customized Message -->
              <div class="form-group">
                <label>Message to Customer:</label>
                <textarea 
                  v-model="customDeclineMessage" 
                  placeholder="Enter a message explaining the changes..." 
                  rows="3"
                ></textarea>
              </div>
              
              <!-- Edit Item Quantities -->
              <div class="form-group">
                <label>Adjust Item Quantities:</label>
                <div class="items-adjustment">
                  <div v-for="(item, index) in editableItems" :key="index" class="item-adjust-row">
                    <span class="item-name">{{ item.name }}</span>
                    <div class="quantity-controls">
                      <button @click="decrementQuantity(index)" :disabled="item.quantity <= 0">-</button>
                      <input type="number" v-model.number="item.quantity" min="0">
                      <button @click="incrementQuantity(index)">+</button>
                    </div>
                    <span class="item-price">₱{{ (item.price * item.quantity).toFixed(2) }}</span>
                  </div>
                </div>
                <p class="adjusted-total"><strong>Adjusted Total:</strong> ₱{{ calculateAdjustedTotal() }}</p>
              </div>
              
              <!-- Action Buttons -->
              <div class="decline-modal-actions">
                <button 
                  @click="sendForApproval" 
                  class="send-approval-btn"
                  :disabled="isUpdating"
                >
                  <span v-if="isUpdating">
                    <i class="fas fa-spinner fa-spin"></i> Updating...
                  </span>
                  <span v-else>Send for Approval</span>
                </button>
                <button 
                  @click="confirmDecline" 
                  class="confirm-decline-btn"
                  :disabled="isUpdating"
                >
                  <span v-if="isUpdating">
                    <i class="fas fa-spinner fa-spin"></i> Declining...
                  </span>
                  <span v-else>Declined</span>
                </button>
              </div>
            </div>
          </div>
        </div>

        <!-- Change Password Modal -->
        <div v-if="showChangePasswordModal" class="password-modal">
          <div class="password-modal-content">
            <div class="password-modal-header">
              <h2>Change Admin Password</h2>
              <button @click="toggleChangePassword" class="close-modal-btn">
                <i class="fa-solid fa-times"></i>
              </button>
            </div>
            <div class="password-modal-body">
              <form @submit.prevent="updatePassword" class="password-form">
                <div class="form-group">
                  <label for="currentPassword">Current Password</label>
                  <input 
                    type="password" 
                    id="currentPassword" 
                    v-model="passwordData.currentPassword" 
                    required
                  />
                </div>
                <div class="form-group">
                  <label for="newPassword">New Password</label>
                  <input 
                    type="password" 
                    id="newPassword" 
                    v-model="passwordData.newPassword" 
                    required
                  />
                </div>
                <div class="form-group">
                  <label for="confirmPassword">Confirm New Password</label>
                  <input 
                    type="password" 
                    id="confirmPassword" 
                    v-model="passwordData.confirmPassword" 
                    required
                  />
                </div>
                <div v-if="passwordMessage" :class="['password-message', passwordMessageType]">
                  {{ passwordMessage }}
                </div>
                <div class="form-actions">
                  <button type="submit" class="save-password-btn">Save New Password</button>
                </div>
              </form>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import ItemEditor from './ItemEditor.vue'

export default {
  components: {
    ItemEditor
  },
  data() {
    return {
      orders: [], // Store pending orders
      isLoading: false, // For loading state
      ws: null, // WebSocket connection
      wsConnected: false,
      activeDeclineOrderId: null, // Track the order for which decline message is being customized
      customDeclineMessage: "", // Store the custom decline message
      notificationSent: false, // To track if the notification has been sent
      searchQuery: "", // To hold the search query input
      isCafeOpen: true,
      notificationMessage: "",
      notificationClass: "", 
      notificationVisible: false,
      showMenuEditor: false, // Control visibility of menu editor popup
      isSidebarOpen: true, // Always open by default
      orderReadyStatus: {}, // Track which orders are ready
      confirmCompleteOrderId: null, // Track which order is being confirmed for completion
      showStockManager: false,
      showStockUpdateModal: false,
      stockItems: [],
      stockSearchQuery: '',
      selectedItem: null,
      stockUpdateAction: 'add',
      stockUpdateQuantity: 0,
      stockUpdateReason: '',
      selectedCategory: '',
      uniqueCategories: [],
      // New properties for decline modal
      showDeclineModal: false,
      selectedOrder: null,
      editableItems: [],
      refreshInterval: null, // To store the interval ID for automatic refresh
      isUpdating: false, // New property for loading state
      pingInterval: null, // To store the interval ID for periodic ping
      showChangePasswordModal: false,
      passwordData: {
        currentPassword: '',
        newPassword: '',
        confirmPassword: ''
      },
      passwordMessage: '',
      passwordMessageType: ''
    };
  },
  computed: {
    // Filter orders based on search query
    filteredOrders() {
      if (!this.searchQuery) {
        return this.orders;
      }
      return this.orders.filter(order => {
        const lowerCaseSearchQuery = this.searchQuery.toLowerCase();
        return (
          order.id.toString().includes(lowerCaseSearchQuery) || // Search by Order ID
          order.customer_name.toLowerCase().includes(lowerCaseSearchQuery) // Search by Customer Name
        );
      });
    },
    filteredStockItems() {
      return this.stockItems.filter(item => {
        if (!item || !item.name) return false;
        const matchesSearch = item.name.toLowerCase().includes((this.stockSearchQuery || '').toLowerCase());
        const matchesCategory = !this.selectedCategory || item.category === this.selectedCategory;
        return matchesSearch && matchesCategory;
      });
    }
  },

  methods: {
    toggleCafeStatus() {
    this.isCafeOpen = !this.isCafeOpen; // Toggle the cafe status
    localStorage.setItem('isCafeOpen', this.isCafeOpen); // Store cafe status

    // Set the notification message and class based on the cafe status
    if (this.isCafeOpen) {
      this.notificationMessage = "Cafe Beàta is now Open!";
      this.notificationClass = "open-notification"; // Set class for green when open
    } else {
      this.notificationMessage = "Cafe Beàta is now Closed!";
      this.notificationClass = "closed-notification"; // Set class for red when closed
    }

    this.showNotification();  // Show the notification
  },
  
   showNotification() {
    // Show the notification and reset visibility after a timeout
    this.notificationVisible = true;

    setTimeout(() => {
      this.notificationVisible = false;
    }, 3000);  // Hide after 3 seconds
  },


    timeAgo(timestamp) {
    // If timestamp is a string, ensure it's in ISO format by replacing space with "T"
    if (typeof timestamp === "string") {
      timestamp = timestamp.replace(" ", "T"); // Convert to ISO format: "YYYY-MM-DD HH:MM:SS" -> "YYYY-MM-DDTHH:MM:SS"
    }

    const now = new Date();
    const orderTime = new Date(timestamp); // Parse the timestamp

    // Check if the timestamp is valid
    if (isNaN(orderTime)) {
      return "Invalid time"; // Return fallback message if timestamp is invalid
    }

    const differenceInSeconds = Math.floor((now - orderTime) / 1000);

    if (differenceInSeconds < 60) {
      return 'Just now';
    } else if (differenceInSeconds < 3600) {
      const minutes = Math.floor(differenceInSeconds / 60);
      return `${minutes} minute${minutes > 1 ? 's' : ''} ago`;
    } else if (differenceInSeconds < 86400) {
      const hours = Math.floor(differenceInSeconds / 3600);
      return `${hours} hour${hours > 1 ? 's' : ''} ago`;
    } else if (differenceInSeconds < 2592000) {
      const days = Math.floor(differenceInSeconds / 86400);
      return `${days} day${days > 1 ? 's' : ''} ago`;
    } else if (differenceInSeconds < 31536000) {
      const months = Math.floor(differenceInSeconds / 2592000);
      return `${months} month${months > 1 ? 's' : ''} ago`;
    } else {
      const years = Math.floor(differenceInSeconds / 31536000);
      return `${years} year${years > 1 ? 's' : ''} ago`;
    }
  },



    // Method to format the order date in the required format
    formatDate(dateString) {
      const date = new Date(dateString);
      const month = (date.getMonth() + 1).toString().padStart(2, '0');
      const day = date.getDate().toString().padStart(2, '0');
      const year = date.getFullYear();
      const hours = date.getHours();
      const minutes = date.getMinutes().toString().padStart(2, '0');
      const period = hours >= 12 ? 'PM' : 'AM';
      const hour12 = (hours % 12 || 12).toString().padStart(2, '0');
      
      // Format: MM-DD-YYYY with highlighted time
      const formattedDate = `${month}-${day}-${year} <span class="highlighted-time">${hour12}:${minutes} ${period}</span>`;
      return formattedDate;
    },

    cancelDecline() {
      this.activeDeclineOrderId = null; // Hide decline input
      this.customDeclineMessage = ""; // Clear text
    },

    // Navigate to the Order Record page
    goToOrderRecord() {
      this.$router.push({ name: "OrderRecord" });  // Ensure this matches the name of the route
    },

    logout() {
      this.$router.push({ name: "Login" });  // Redirect the user to the Login page (adjust the route as needed)
    },

    // Fetch orders only once at initial load
    async fetchOrders() {
      if (this.isLoading) return;
      this.isLoading = true;
      
      try {
        const response = await fetch("http://127.0.0.1:8000/orders");
        const data = await response.json();
        if (data.orders && Array.isArray(data.orders)) {
          // Filter pending orders and sort them by ID in ascending order
          const pendingOrders = data.orders
            .filter(order => order.status === "pending")
            .sort((a, b) => {
              // Convert order IDs to numbers for proper numerical sorting
              const idA = parseInt(a.id);
              const idB = parseInt(b.id);
              return idA - idB; // Sort in ascending order (lower IDs first)
            });
          
          console.log(`Fetched ${pendingOrders.length} pending orders`);
          
          // Important: Create a new array reference to ensure Vue reactivity
          this.orders = [...pendingOrders];
          
          // Check if any of the fetched orders have ready notifications
          // This ensures the "Mark as Completed" button is enabled for orders that are ready
          this.orders.forEach(order => {
            // If the order is already marked as ready in localStorage, keep that status
            if (this.orderReadyStatus[order.id]) {
              return;
            }
            
            // Check if there's a notification for this order
            const userNotificationsKey = `user_notifications_${order.customer_name}`;
            const notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
            const hasReadyNotification = notifications.some(n => n.orderId === order.id);
            
            if (hasReadyNotification) {
              this.orderReadyStatus[order.id] = true;
            }
          });
          
          // Update localStorage with any changes
          localStorage.setItem('orderReadyStatus', JSON.stringify(this.orderReadyStatus));
          
          // Force the UI to update
          this.$forceUpdate();
        } else {
          console.error("Invalid data format", data);
          this.orders = [];
        }
      } catch (error) {
        console.error("Error fetching orders:", error);
      } finally {
        this.isLoading = false;
      }
    },

    // Format ordered items for notification message
    formatItems(items) {
      if (!Array.isArray(items)) {
        console.error("Invalid item format:", items);
        return "Invalid item data";
      }
      return items.map(item => `${item.name} x${item.quantity}`).join(", ");
    },

    // Calculate the total price for a single order
    calculateOrderTotal(items) {
      if (!Array.isArray(items)) return "₱0";
      return items.reduce((sum, item) => sum + item.price * item.quantity, 0).toFixed(2);
    },

    // Mark an order as completed and send notification
    markAsCompleted(orderId, customerName, items) {
      // Log for debugging
      console.log(`Marking order ${orderId} as completed...`);
      
      // Ensure items is properly formatted
      let processedItems = items;
      if (typeof items === 'string') {
        try {
          processedItems = JSON.parse(items);
        } catch (e) {
          console.error("Failed to parse items:", e);
          alert("Error processing order items. Please try again.");
          return;
        }
      }
      
      fetch(`http://127.0.0.1:8000/orders/${orderId}`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ status: "completed" })
      })
        .then(response => {
          // First check if the response is actually received
          if (!response) {
            throw new Error('No response received from server');
          }
          
          // Then check if it's OK
          if (!response.ok) {
            return response.text().then(text => {
              try {
                // Try to parse as JSON
                const data = JSON.parse(text);
                throw new Error(data.detail || `Server error: ${response.status}`);
              } catch (e) {
                // If parsing fails, use the raw text
                throw new Error(`Server error: ${response.status} - ${text || 'Unknown error'}`);
              }
            });
          }
          return response.json();
        })
        .then((data) => {
          console.log("Order completed successfully:", data);
          
          // Immediately remove from pending orders
          this.orders = this.orders.filter(order => order.id !== orderId);

          // Remove from orderReadyStatus
          delete this.orderReadyStatus[orderId];
          // Update localStorage
          localStorage.setItem('orderReadyStatus', JSON.stringify(this.orderReadyStatus));

          // Calculate the total price
          const total = processedItems.reduce((sum, item) => sum + item.price * item.quantity, 0).toFixed(2);

          // Prepare the notification with highlighted order details
          const notification = {
            orderId,
            customerName,
            message: `Your order is completed! ✔️ Thank you for choosing Café Beata. Enjoy your food and drinks! 🥰. <span class="highlighted-order-details">Order details: ${this.formatItems(processedItems)}. Total: ₱${total}</span>`,
            timestamp: new Date().toISOString(),
            items: processedItems,
            total,
          };

          // Save the notification in localStorage for the specific user
          const userNotificationsKey = `user_notifications_${customerName}`;
          let notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
          notifications.push(notification);
          localStorage.setItem(userNotificationsKey, JSON.stringify(notifications));

          // Send real-time notification via WebSocket if connected
          if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify({
              type: 'user_notification',
              action: 'order_completed',
              notification: notification,
              target_user: customerName
            }));
          }

          // Emit an event to notify other components
          window.dispatchEvent(new Event("notificationUpdated"));

          alert("Order marked as completed!");
        })
        .catch(error => {
          console.error("Error marking order as completed:", error);
          alert(error.message || "Error completing order. Please try again.");
          // Refresh orders to ensure UI is in sync
          this.fetchOrders();
        });
    },

    // Open the decline modal for a specific order
    openDeclineDialog(order) {
      this.activeDeclineOrderId = order.id;
      this.selectedOrder = order;
      this.customDeclineMessage = localStorage.getItem(`customDeclineMessage_${order.id}`) || "We need to adjust your order due to availability issues.";
      
      // Clone the items array to make it editable
      this.editableItems = JSON.parse(JSON.stringify(order.items));
      
      // Show the modal
      this.showDeclineModal = true;
    },

    // Close the decline modal
    closeDeclineModal() {
      this.showDeclineModal = false;
      this.activeDeclineOrderId = null;
      this.selectedOrder = null;
    },

    // Increment item quantity
    incrementQuantity(index) {
      this.editableItems[index].quantity++;
    },

    // Decrement item quantity
    decrementQuantity(index) {
      if (this.editableItems[index].quantity > 0) {
        this.editableItems[index].quantity--;
      }
    },

    // Calculate the total of adjusted items
    calculateAdjustedTotal() {
      return this.editableItems.reduce((sum, item) => sum + (item.price * item.quantity), 0).toFixed(2);
    },

    // Send order adjustment for customer approval
    sendForApproval() {
      if (!this.selectedOrder) return;
      
      const orderId = this.selectedOrder.id;
      const customerName = this.selectedOrder.customer_name;
      
      // Calculate the total price
      const total = this.calculateAdjustedTotal();
      
      // Format items for display
      const formattedItems = this.formatItems(this.editableItems);
      
      // Build a more specific message about quantity adjustments
      let adjustmentReasons = [];
      this.editableItems.forEach((item) => {
        const originalItem = this.selectedOrder.items.find(i => i.name === item.name);
        if (originalItem && originalItem.quantity !== item.quantity) {
          adjustmentReasons.push(`${item.name} adjusted from ${originalItem.quantity} to ${item.quantity} due to limited availability`);
        }
      });
      
      // Create the customized message with specific quantity adjustment reasons
      const specificAdjustments = adjustmentReasons.length > 0 
        ? `The following adjustments were made: ${adjustmentReasons.join('; ')}. ` 
        : '';
        
      // Prepare notification message with approval buttons
      const message = `${this.customDeclineMessage} ${specificAdjustments}Please review the adjusted order: ${formattedItems}. Total: ₱${total}`;
      
      // Create adjustment notification with approval options
      const notification = {
        orderId,
        customerName,
        message,
        timestamp: new Date().toISOString(),
        items: this.editableItems,
        total,
        requiresApproval: true,  // Flag to indicate this needs user approval
        originalItems: this.selectedOrder.items  // Store original items for reference
      };
      
      // Set loading state
      this.isUpdating = true;
      
      // First update the order in the database with the adjusted quantities
      fetch(`http://127.0.0.1:8000/orders/${orderId}/update-items`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          items: this.editableItems,
          status: "pending",  // Keep status as pending
          isPendingApproval: true // Add flag for pending approval
        })
      })
      .then(response => {
        if (!response.ok) {
          return response.text().then(text => {
            console.error(`Error status: ${response.status}, details: ${text}`);
            throw new Error(`Failed to update order (${response.status}): ${text}`);
          });
        }
        return response.json();
      })
      .then((data) => {
        console.log("Order items updated in database successfully", data);
        
        // Now save to user's notifications
        const userNotificationsKey = `user_notifications_${customerName}`;
        let notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
        notifications.push(notification);
        localStorage.setItem(userNotificationsKey, JSON.stringify(notifications));
        
        // Send WebSocket notification if connected
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
          this.ws.send(JSON.stringify({
            type: 'user_notification',
            action: 'order_adjustment',  // Set specific action type for adjustment requests
            notification: notification,
            target_user: customerName
          }));
        }
        
        // Update the order in the UI immediately to show adjusted quantities
        const orderIndex = this.orders.findIndex(o => o.id === orderId);
        if (orderIndex !== -1) {
          // Create a copy with adjusted items
          const updatedOrder = {
            ...this.orders[orderIndex],
            items: this.editableItems,
            isPendingApproval: true // Add flag for styling
          };
          
          // Update the order in the list - using Vue 3 array mutation
          this.orders = [
            ...this.orders.slice(0, orderIndex),
            updatedOrder,
            ...this.orders.slice(orderIndex + 1)
          ];
        }
        
        // Show confirmation
        this.notificationMessage = "Adjustment request sent to customer for approval!";
        this.notificationClass = "success-notification";
        this.showNotification();
        
        // Close the modal and reset loading state
        this.isUpdating = false;
        this.closeDeclineModal();
      })
      .catch(error => {
        console.error("Error updating order items:", error);
        
        // Reset loading state but keep modal open for retry
        this.isUpdating = false;
        
        // Show error notification
        this.notificationMessage = `Error: ${error.message}`;
        this.notificationClass = "error-notification";
        this.showNotification();
        
        // Ask user if they want to retry via modal or notification
        if (confirm(`Failed to update order: ${error.message}. Do you want to retry?`)) {
          // Retry the update
          this.sendForApproval();
        }
      });
    },

    // Decline the order directly
    confirmDecline() {
      if (!this.selectedOrder) return;
      
      // Show loading indicator
      this.isUpdating = true;
      
      const orderId = this.selectedOrder.id;
      const customerName = this.selectedOrder.customer_name;
      const items = this.selectedOrder.items;
      const message = this.customDeclineMessage || "Unfortunately, this item is temporarily out of stock. We apologize for the inconvenience and appreciate your patience. 🙏";
      
      fetch(`http://127.0.0.1:8000/orders/${orderId}`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ status: "declined" })
      })
      .then(response => {
        if (!response.ok) {
          throw new Error(`HTTP error! Status: ${response.status}`);
        }
        return response.json();
      })
      .then(() => {
        // Remove from pending orders immediately
        this.orders = this.orders.filter(order => order.id !== orderId);
        
        // Remove from orderReadyStatus if it exists
        if (this.orderReadyStatus[orderId]) {
          delete this.orderReadyStatus[orderId];
          localStorage.setItem('orderReadyStatus', JSON.stringify(this.orderReadyStatus));
        }
        
        // Calculate total
        const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0).toFixed(2);
        
        // Prepare notification
        const notification = {
          orderId,
          customerName,
          message: `${message} Order details: ${this.formatItems(items)}. Total: ₱${total}`,
          timestamp: new Date().toISOString(),
          items, 
          total,
        };
        
        // Save notification to localStorage
        const userNotificationsKey = `user_notifications_${customerName}`;
        let notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
        notifications.push(notification);
        localStorage.setItem(userNotificationsKey, JSON.stringify(notifications));
        
        // Send direct WebSocket notification for real-time updates
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
          console.log('Sending WebSocket notification for declined order:', orderId);
          
          // First send the standard notification
          this.ws.send(JSON.stringify({
            type: 'user_notification',
            action: 'order_declined',
            notification: notification,
            target_user: customerName
          }));
          
          // Then send a special order_declined type message for real-time notification
          this.ws.send(JSON.stringify({
            type: 'order_declined',
            order_id: orderId,
            customer_name: customerName,
            reason: message,
            timestamp: new Date().toISOString()
          }));
        }
        
        // Reset loading state
        this.isUpdating = false;
        
        // Close the modal
        this.closeDeclineModal();
        
        // Show success message
        this.notificationMessage = `Order #${orderId} has been declined`;
        this.notificationClass = "closed-notification";
        this.showNotification();
        
      })
      .catch(error => {
        console.error("Error declining order:", error);
        this.isUpdating = false;
        alert("Error declining order. Please try again.");
      });
    },

    // Original decline order function - unchanged
    declineOrder(orderId, customerName, items) {
      const message = this.customDeclineMessage || "Unfortunately, this item is temporarily out of stock. We apologize for the inconvenience and appreciate your patience. 🙏";

      fetch(`http://127.0.0.1:8000/orders/${orderId}`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ status: "declined" }) // Properly formatted JSON
      })
        .then(response => {
          if (!response.ok) {
            throw new Error(`HTTP error! Status: ${response.status}`);
          }
          return response.json();
        })
        .then(() => {
          // Immediately remove from pending orders
          this.orders = this.orders.filter(order => order.id !== orderId);

          // Remove from orderReadyStatus if it exists
          if (this.orderReadyStatus[orderId]) {
            delete this.orderReadyStatus[orderId];
            // Update localStorage
            localStorage.setItem('orderReadyStatus', JSON.stringify(this.orderReadyStatus));
          }

          // Calculate the total price
          const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0).toFixed(2);

          // Prepare the notification with the custom message and order details
          const notification = {
            orderId,
            customerName,
            message: `${message} Order details: ${this.formatItems(items)}. Total: ₱${total}`,
            timestamp: new Date().toISOString(),
            items,  // Include items in the notification
            total,  // Include total in the notification
          };

          // Save the notification in localStorage for the specific user
          const userNotificationsKey = `user_notifications_${customerName}`;
          let notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
          notifications.push(notification);
          localStorage.setItem(userNotificationsKey, JSON.stringify(notifications));

          // Send direct WebSocket notification for real-time updates
          if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            console.log('Sending WebSocket notification for declined order:', orderId);
            
            // First send the standard notification
            this.ws.send(JSON.stringify({
              type: 'user_notification',
              action: 'order_declined',
              notification: notification,
              target_user: customerName
            }));
            
            // Then send a special order_declined type message for real-time notification
            this.ws.send(JSON.stringify({
              type: 'order_declined',
              order_id: orderId,
              customer_name: customerName,
              reason: message,
              timestamp: new Date().toISOString()
            }));
          }

          // Emit an event to notify other components (optional)
          window.dispatchEvent(new Event("orderDeclined"));

          // Show success message
          this.notificationMessage = `Order #${orderId} has been declined`;
          this.notificationClass = "closed-notification";
          this.showNotification();
        })
        .catch(error => {
          console.error("Error declining order:", error);
          this.isUpdating = false;
          alert("Error declining order. Please try again.");
        });
    },

    // Save the decline message to localStorage whenever it's updated
    updateDeclineMessage() {
      if (this.activeDeclineOrderId !== null) {
        localStorage.setItem(`customDeclineMessage_${this.activeDeclineOrderId}`, this.customDeclineMessage);
      }
    },

    // New method to handle the "Order Ready" button click and show pop-up notification
    sendOrderReadyNotification(orderId, customerName, items) {
      const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0).toFixed(2);

      const notification = {
        orderId,
        customerName,
        message: `Your order is now ready! Proceed to the cashier for payment and pickup. ☺️ Order details: ${this.formatItems(items)}. Total: ₱${total}`,
        timestamp: new Date().toISOString(),
        items,
        total,
      };

      // Save the notification in localStorage for the specific user
      const userNotificationsKey = `user_notifications_${customerName}`;
      let notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
      
      // Add the notification without replacing existing ones
      notifications.push(notification);
      
      // Sort notifications by timestamp (newest first)
      notifications.sort((a, b) => {
        const dateA = new Date(a.timestamp);
        const dateB = new Date(b.timestamp);
        return dateB - dateA;
      });
      
      localStorage.setItem(userNotificationsKey, JSON.stringify(notifications));

      // Set order as ready using direct assignment
      this.orderReadyStatus[orderId] = true;
      // Force reactivity update
      this.orderReadyStatus = { ...this.orderReadyStatus };
      
      // Save orderReadyStatus to localStorage
      localStorage.setItem('orderReadyStatus', JSON.stringify(this.orderReadyStatus));

      // Send real-time notification via WebSocket if connected
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({
          type: 'user_notification',
          action: 'order_ready',
          notification: notification,
          target_user: customerName
        }));
      }

      // Show success notification to admin
      this.notificationSent = true;
      setTimeout(() => {
        this.notificationSent = false;
      }, 3000);
    },

    // Toggle menu editor popup visibility
    toggleMenuEditor() {
      this.showMenuEditor = !this.showMenuEditor;
      
      // When opening the modal, prevent scrolling on the body
      if (this.showMenuEditor) {
        document.body.style.overflow = 'hidden';
      } else {
        document.body.style.overflow = '';
      }
    },

    // Toggle sidebar visibility
    toggleSidebar() {
      this.isSidebarOpen = !this.isSidebarOpen;
      // When opening the sidebar, prevent scrolling on the body
      if (this.isSidebarOpen) {
        document.body.style.overflow = 'hidden';
      } else {
        document.body.style.overflow = '';
      }
    },

    // Close sidebar - only called when X button is clicked
    closeSidebar() {
      this.isSidebarOpen = false;
      document.body.style.overflow = '';
    },

    // Handle clicks outside sidebar - removed to prevent auto-closing when clicking outside
    handleOutsideClick() {
      // Do nothing - sidebar should stay open
    },

    // Add new methods for completion confirmation
    showCompletionConfirmation(orderId) {
      this.confirmCompleteOrderId = orderId;
    },

    confirmCompletion() {
      const order = this.orders.find(o => o.id === this.confirmCompleteOrderId);
      if (order) {
        this.markAsCompleted(order.id, order.customer_name, order.items);
      }
      this.confirmCompleteOrderId = null;
    },

    cancelCompletion() {
      this.confirmCompleteOrderId = null;
    },

    toggleStockManager() {
      this.showStockManager = !this.showStockManager;
      if (this.showStockManager) {
        this.fetchStockItems();
      }
    },

    async fetchStockItems() {
      try {
        const response = await fetch('http://localhost:8000/api/stocks');
        const data = await response.json();
        console.log('Fetched stock data:', data); // Debug log
        
        if (data.success && Array.isArray(data.items)) {
          // Map the items to include name and category from item_name
          this.stockItems = data.items.map(item => ({
            id: item.item_id, // Use item_id from the API response
            name: item.item_name,
            category: item.category,
            quantity: item.quantity,
            min_stock_level: item.min_stock_level
          }));
          
          // Update unique categories
          this.uniqueCategories = [...new Set(this.stockItems.map(item => item.category))];
          console.log('Processed stock items:', this.stockItems); // Debug log
        } else {
          console.error('Invalid data format received:', data);
        }
      } catch (error) {
        console.error('Error fetching stock items:', error);
      }
    },

    getStockStatus(item) {
      if (item.quantity === 0) return 'Disabled (Out of Stock)';
      if (item.quantity >= 999999) return 'Enabled (Unlimited)';
      if (item.quantity <= item.min_stock_level) return 'Low Stock';
      return 'In Stock';
    },

    getStockStatusClass(item) {
      if (item.quantity === 0) return 'status-disabled';
      if (item.quantity >= 999999) return 'status-enabled';
      if (item.quantity <= item.min_stock_level) return 'status-low';
      return 'status-ok';
    },

    openStockUpdateModal(item) {
      console.log('Opening modal for item:', item); // Debug log
      this.selectedItem = item;
      this.showStockUpdateModal = true;
      this.stockUpdateQuantity = 0;
      this.stockUpdateAction = 'add';
      this.stockUpdateReason = '';
    },

    closeStockUpdateModal() {
      this.showStockUpdateModal = false;
      this.selectedItem = null;
      this.stockUpdateQuantity = 0;
      this.stockUpdateReason = '';
    },

    async submitStockUpdate() {
      // Validate required fields
      if (!this.selectedItem || !this.selectedItem.id) {
        alert('No item selected');
        return;
      }

      if (!this.stockUpdateAction) {
        alert('Please select an action');
        return;
      }

      try {
        let requestBody = {};
        
        // Handle different action types
        if (this.stockUpdateAction === 'disabled') {
          // For disabled, set quantity to 0 and use 'set' action
          requestBody = {
            action: 'set',
            quantity: 0,
            reason: this.stockUpdateReason || 'Disabled - Out of Stock'
          };
        } else if (this.stockUpdateAction === 'enabled') {
          // For enabled, set a special value to indicate unlimited
          requestBody = {
            action: 'set',
            quantity: 999999, // Very large number to represent unlimited
            reason: this.stockUpdateReason || 'Enabled - Unlimited Orders'
          };
        } else {
          // For regular actions (add, subtract, set)
          if (!this.stockUpdateQuantity || this.stockUpdateQuantity <= 0) {
            alert('Please enter a valid quantity (greater than 0)');
            return;
          }

          // Additional validation for subtract action
          if (this.stockUpdateAction === 'subtract' && this.stockUpdateQuantity > this.selectedItem.quantity) {
            alert('Cannot remove more than current stock');
            return;
          }
          
          requestBody = {
            action: this.stockUpdateAction,
            quantity: parseInt(this.stockUpdateQuantity),
            reason: this.stockUpdateReason || 'Stock update'
          };
        }

        console.log('Sending request:', requestBody);

        const response = await fetch(`http://localhost:8000/api/stocks/${this.selectedItem.id}/update`, {
          method: 'PUT',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(requestBody)
        });

        const data = await response.json();
        
        if (!response.ok) {
          throw new Error(data.detail || 'Failed to update stock');
        }

        if (data.success) {
          alert('Stock updated successfully!');
          this.closeStockUpdateModal();
          await this.fetchStockItems(); // Refresh the stock list
        } else {
          throw new Error(data.message || 'Failed to update stock');
        }
      } catch (error) {
        console.error('Error updating stock:', error);
        alert(error.message || 'Failed to update stock. Please try again.');
      }
    },

    initWebSocket() {
      // Use the same host as the API
      const wsUrl = `ws://${window.location.hostname}:8000/ws/orders`;
      
      // Close existing connection if it exists
      if (this.ws) {
        try {
          this.ws.close();
        } catch (e) {
          console.error("Error closing existing WebSocket:", e);
        }
      }
      
      console.log('Initializing WebSocket connection...');
      this.ws = new WebSocket(wsUrl);
      
      this.ws.onopen = () => {
        console.log('WebSocket connected');
        this.wsConnected = true;
        
        // Set up a periodic ping to keep the connection alive
        if (this.pingInterval) clearInterval(this.pingInterval);
        this.pingInterval = setInterval(() => {
          if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify({ type: 'ping' }));
          }
        }, 30000); // ping every 30 seconds
      };
      
      this.ws.onmessage = async (event) => {
        try {
          const data = JSON.parse(event.data);
          console.log('WebSocket message received:', data);

          if (data.type === 'new_order') {
            // Handle new order
            if (data.order.status === 'pending') {
              // Add the new order to the orders array
              this.orders.push(data.order);
              
              // Re-sort the orders array by ID in ascending order
              this.orders.sort((a, b) => {
                const idA = parseInt(a.id);
                const idB = parseInt(b.id);
                return idA - idB; // Sort in ascending order (lower IDs first)
              });
            }
          } else if (data.type === 'order_status_update') {
            // Handle order status update
            if (data.status !== 'pending') {
              this.orders = this.orders.filter(order => order.id !== data.order_id);
              
              // Remove from orderReadyStatus if it exists
              if (this.orderReadyStatus[data.order_id]) {
                delete this.orderReadyStatus[data.order_id];
                // Update localStorage
                localStorage.setItem('orderReadyStatus', JSON.stringify(this.orderReadyStatus));
              }
            }
          } else if (data.type === 'admin_notification') {
            // Handle customer responses to order adjustments
            if (data.action === 'adjustment_response') {
              // Process the customer's response to an order adjustment
              const notification = data.notification;
              const orderId = notification.orderId;
              
              if (notification.message.includes('APPROVED')) {
                // Find the order and update it with adjusted items
                const orderIndex = this.orders.findIndex(o => o.id === orderId);
                if (orderIndex !== -1) {
                  // Update the order with the adjusted items and remove pending approval flag
                  const updatedOrder = {
                    ...this.orders[orderIndex],
                    items: notification.items,
                    isPendingApproval: false // Remove the pending approval flag
                  };
                  
                  // Update using Vue 3 reactivity 
                  this.orders = [
                    ...this.orders.slice(0, orderIndex),
                    updatedOrder,
                    ...this.orders.slice(orderIndex + 1)
                  ];
                  
                  // Show success notification
                  this.notificationMessage = `Customer has approved order adjustments for Order #${orderId}`;
                  this.notificationClass = "open-notification";
                  this.showNotification();
                }
              } else if (notification.message.includes('DECLINED')) {
                // Remove the order from the list if customer declined
                this.orders = this.orders.filter(o => o.id !== orderId);
                
                // Show declined notification
                this.notificationMessage = `Customer has declined order adjustments for Order #${orderId}`;
                this.notificationClass = "closed-notification";
                this.showNotification();
              }
              
              // Mark the notification as processed
              notification.processed = true;
              
              // Update the notification in localStorage
              const adminNotificationsKey = 'user_notifications_Admin';
              let adminNotifications = JSON.parse(localStorage.getItem(adminNotificationsKey)) || [];
              
              // Find and update the processed notification
              const notificationIndex = adminNotifications.findIndex(n => 
                n.isAdminNotification && n.orderId === orderId && !n.processed
              );
              
              if (notificationIndex !== -1) {
                adminNotifications[notificationIndex].processed = true;
                localStorage.setItem(adminNotificationsKey, JSON.stringify(adminNotifications));
              }
            }
          } else if (data.type === 'customer_approval') {
            // Direct customer approval handling for real-time updates
            const { orderId, approved } = data;
            
            if (approved) {
              console.log(`CRITICAL FIX: Customer approved order #${orderId} - handling the single-order case specially`);
              
              // Check if this is the single-order case (the problematic case)
              const isSingleOrder = this.orders.length === 1;
              
              // Always try direct DOM manipulation first
              try {
                const orderElement = document.querySelector(`.order-item[data-order-id="${orderId}"]`);
                if (orderElement) {
                  console.log("Found order element in DOM, applying direct DOM updates");
                  
                  // Remove the pending approval class and add all styling inline
                  orderElement.classList.remove('order-pending-approval');
                  orderElement.style.backgroundColor = "#ffffff";
                  orderElement.style.border = "2px solid #ddd";
                  orderElement.style.boxShadow = "0 2px 5px rgba(0, 0, 0, 0.1)";
                  
                  // Hide the PENDING APPROVAL label
                  const style = document.createElement('style');
                  style.id = `fix-order-${orderId}`;
                  style.textContent = `
                    .order-item[data-order-id="${orderId}"]::before {
                      display: none !important;
                    }
                    .order-item[data-order-id="${orderId}"] h3 {
                      color: #333 !important;
                    }
                    .order-item[data-order-id="${orderId}"] .order-total {
                      color: #333 !important;
                    }
                  `;
                  document.head.appendChild(style);
                  
                  // Update the status text
                  const statusParagraphs = orderElement.querySelectorAll('p');
                  for (const paragraph of statusParagraphs) {
                    if (paragraph.innerHTML.includes('Status:')) {
                      paragraph.innerHTML = '<strong>Status:</strong> pending';
                      break;
                    }
                  }
                }
                
                // For the single-order case, recreate the entire orders container
                if (isSingleOrder) {
                  console.log("Single order detected - using special handling");
                  
                  // 1. Get the order container
                  const ordersContainer = document.querySelector('.orders-container');
                  if (!ordersContainer) {
                    console.error("Orders container not found");
                    return;
                  }
                  
                  // 2. Find the order in our data
                  const order = this.orders.find(o => o.id === orderId);
                  if (!order) {
                    console.error("Order not found in data");
                    return;
                  }
                  
                  // 3. Create an updated version of the order without pending approval
                  const updatedOrder = { ...order, isPendingApproval: false };
                  
                  // 4. Replace the entire orders list HTML
                  const ordersList = ordersContainer.querySelector('.orders-list');
                  if (!ordersList) {
                    console.error("Orders list not found");
                    return;
                  }
                  
                  // Create a new order item element 
                  const newOrderItemHTML = `
                    <div class="order-item" data-order-id="${updatedOrder.id}">
                      <div class="order-details">
                        <h3>Order ID: ${updatedOrder.id}</h3>
                        <p><strong>Customer:</strong> ${updatedOrder.customer_name}</p>
                        <p><strong>Status:</strong> ${updatedOrder.status}</p>
                        <p><strong>Time Order:</strong> ${this.timeAgo(updatedOrder.created_at)}</p>
                        
                        <div class="items-section">
                          <strong>Items:</strong>
                          <ul>
                            ${updatedOrder.items.map(item => 
                              `<li>${item.name} - ₱${item.price} x ${item.quantity}</li>`
                            ).join('')}
                          </ul>
                        </div>
                        
                        <div class="order-total">
                          <p><strong>Total Amount: ₱${this.calculateOrderTotal(updatedOrder.items)}</strong></p>
                        </div>
                      </div>
                      
                      <div class="order-actions">
                        <button 
                          onclick="document.dispatchEvent(new CustomEvent('completion-confirmation', {detail: {orderId: ${updatedOrder.id}}}))"
                          class="mark-completed-btn small-btn ${!this.orderReadyStatus[updatedOrder.id] ? 'disabled' : ''}"
                          ${!this.orderReadyStatus[updatedOrder.id] ? 'disabled' : ''}
                        >
                          Mark as Completed
                        </button>
                        
                        <button 
                          onclick="document.dispatchEvent(new CustomEvent('order-ready', {detail: {orderId: ${updatedOrder.id}, customerName: '${updatedOrder.customer_name}'}}))"
                          class="order-ready-btn small-btn"
                        >
                          Order Ready 🔔
                        </button>
                        
                        <button 
                          onclick="document.dispatchEvent(new CustomEvent('decline-dialog', {detail: {orderId: ${updatedOrder.id}}}))"
                          class="decline-btn"
                        >
                          Decline
                        </button>
                      </div>
                    </div>
                  `;
                  
                  // Replace the content
                  ordersList.innerHTML = newOrderItemHTML;
                  
                  // Setup event listeners for our custom events
                  if (!this._customListenersAdded) {
                    document.addEventListener('completion-confirmation', (e) => {
                      this.showCompletionConfirmation(e.detail.orderId);
                    });
                    
                    document.addEventListener('order-ready', (e) => {
                      this.sendOrderReadyNotification(e.detail.orderId, e.detail.customerName, updatedOrder.items);
                    });
                    
                    document.addEventListener('decline-dialog', (e) => {
                      // Find the order again since we need the full object
                      const order = this.orders.find(o => o.id === e.detail.orderId);
                      if (order) {
                        this.openDeclineDialog(order);
                      }
                    });
                    
                    this._customListenersAdded = true;
                  }
                  
                  // Also update our Vue data model
                  this.orders = [updatedOrder];
                }
              } catch (error) {
                console.error("Error during direct DOM manipulation:", error);
              }
              
              // Always show notification to admin
              this.notificationMessage = `The customer has APPROVED the order adjustments. Order ID: ${orderId}`;
              this.notificationClass = "open-notification";
              this.showNotification();
              
              // Update the database in the background
              fetch(`http://127.0.0.1:8000/orders/${orderId}/update-items`, {
                method: "PUT",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({
                  status: "pending",
                  isPendingApproval: false
                })
              })
              .then(response => response.json())
              .then(data => {
                console.log("Database updated to reflect customer approval:", data);
                
                // Always fetch all orders as a fallback
                setTimeout(() => {
                  this.fetchPendingOrders();
                }, 1000);
              })
              .catch(err => {
                console.error("Error updating database after customer approval:", err);
                
                // Force reload as last resort for error cases
                if (isSingleOrder) {
                  setTimeout(() => location.reload(), 1000);
                }
              });
            } else {
              // If declined, handle in Vue since that part seems to work
              this.orders = this.orders.filter(o => o.id !== orderId);
              
              // Show notification about the decline
              this.notificationMessage = `Customer has declined order adjustments for Order #${orderId}`;
              this.notificationClass = "closed-notification";
              this.showNotification();
            }
          } else if (data.type === 'stock_update') {
            // Handle stock update
            const stockItem = this.stockItems.find(item => item.id === data.item_id);
            if (stockItem) {
              stockItem.quantity = data.new_quantity;
              stockItem.min_stock_level = data.min_stock_level;
              
              // Update unique categories if needed
              if (!this.uniqueCategories.includes(data.category)) {
                this.uniqueCategories.push(data.category);
              }
            }
            // Refresh stock items to ensure all data is up to date
            await this.fetchStockItems();
          } else if (data.type === 'menu_update') {
            // Handle menu updates (new items, edited items, or deleted items)
            await this.fetchStockItems(); // Refresh stock items when menu changes
          } else if (data.type === 'category_update') {
            // Handle category updates
            await this.fetchStockItems(); // Refresh stock items when categories change
            
            // Update unique categories list
            if (data.action === 'add' && data.category && data.category.name) {
              if (!this.uniqueCategories.includes(data.category.name)) {
                this.uniqueCategories.push(data.category.name);
              }
            } else if (data.action === 'update' && data.category) {
              // Replace old category name with new one
              const index = this.uniqueCategories.indexOf(data.category.old_name);
              if (index !== -1) {
                this.uniqueCategories[index] = data.category.name;
              } else if (!this.uniqueCategories.includes(data.category.name)) {
                this.uniqueCategories.push(data.category.name);
              }
              
              // Update selected category if it was renamed
              if (this.selectedCategory === data.category.old_name) {
                this.selectedCategory = data.category.name;
              }
            } else if (data.action === 'delete' && data.category_name) {
              // Remove deleted category
              this.uniqueCategories = this.uniqueCategories.filter(cat => cat !== data.category_name);
              
              // Reset selected category if it was deleted
              if (this.selectedCategory === data.category_name) {
                this.selectedCategory = '';
              }
            }
          }
        } catch (error) {
          console.error('Error processing WebSocket message:', error);
        }
      };
      
      this.ws.onclose = () => {
        console.log('WebSocket disconnected');
        this.wsConnected = false;
        // Try to reconnect after 5 seconds
        setTimeout(() => {
          this.initWebSocket();
        }, 5000);
      };
      
      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        this.wsConnected = false;
      };
    },

    // Handle localStorage changes for admin notifications
    handleStorageEvent(event) {
      // Check if the storage event is for admin notifications
      if (event.key === 'user_notifications_Admin') {
        // Process new admin notifications
        this.processAdminNotifications();
      }
    },
    
    // Process admin notifications from localStorage
    processAdminNotifications() {
      // Get admin notifications 
      const adminNotificationsKey = 'user_notifications_Admin';
      const adminNotifications = JSON.parse(localStorage.getItem(adminNotificationsKey)) || [];
      
      // Check for unprocessed customer response notifications
      const unprocessedNotifications = adminNotifications.filter(n => 
        n.isAdminNotification && 
        (n.message.includes('APPROVED') || n.message.includes('DECLINED')) && 
        !n.processed
      );
      
      // Process each notification
      unprocessedNotifications.forEach(notification => {
        const orderId = notification.orderId;
        
        if (notification.message.includes('APPROVED')) {
          // If customer approved adjustments
          // Find the order and update it
          const orderIndex = this.orders.findIndex(o => o.id === orderId);
          
          if (orderIndex !== -1) {
            // Update the order with adjusted items
            this.orders[orderIndex].items = notification.items;
            
            // Mark the notification as processed
            notification.processed = true;
          }
        } else if (notification.message.includes('DECLINED')) {
          // If customer declined, remove the order
          this.orders = this.orders.filter(o => o.id !== orderId);
          
          // Mark the notification as processed
          notification.processed = true;
        }
      });
      
      // Update the processed notifications in localStorage
      localStorage.setItem(adminNotificationsKey, JSON.stringify(adminNotifications));
      
      // Show notification to admin about the response
      if (unprocessedNotifications.length > 0) {
        const lastNotification = unprocessedNotifications[unprocessedNotifications.length - 1];
        this.notificationMessage = lastNotification.message;
        this.notificationClass = lastNotification.message.includes('APPROVED') ? 
          'open-notification' : 'closed-notification';
        this.showNotification();
      }
    },
    
    // Fetch pending orders - wrapper for fetchOrders for consistency
    fetchPendingOrders() {
      this.fetchOrders();
    },
    
    // Connect to WebSocket - renamed from initWebSocket for consistency
    connectWebSocket() {
      this.initWebSocket();
    },

    // Add this method to force refresh orders after WebSocket updates
    forceRefreshOrder(orderId) {
      console.log(`Forcing refresh for order #${orderId}, current orders count: ${this.orders.length}`);
      
      // Find the order that needs refreshing
      const orderIndex = this.orders.findIndex(o => o.id === orderId);
      
      if (orderIndex !== -1) {
        // Fetch the latest order data from the server to ensure it's up to date
        fetch(`http://127.0.0.1:8000/orders/${orderId}`)
          .then(response => response.json())
          .then(data => {
            if (data) {
              console.log('Fetched fresh order data for real-time update:', data);
              
              // Create a fresh order object with the latest data
              const updatedOrder = {
                ...data,
                isPendingApproval: data.isPendingApproval || false // Ensure the flag is set correctly
              };
              
              // Special handling for single order
              if (this.orders.length === 1) {
                this.orders = [updatedOrder];
              } else {
                // Replace the order in the array using Vue reactivity
                this.orders = [
                  ...this.orders.slice(0, orderIndex),
                  updatedOrder,
                  ...this.orders.slice(orderIndex + 1)
                ];
              }
              
              // Force the component to re-render
              this.$forceUpdate();
              
              console.log('Order updated successfully in UI:', updatedOrder);
              
              // If order is approved but UI doesn't update, try refreshing all orders
              if (!updatedOrder.isPendingApproval) {
                setTimeout(() => {
                  this.fetchPendingOrders();
                }, 300);
              }
            }
          })
          .catch(error => {
            console.error('Error refreshing order:', error);
            // Fallback to refreshing all orders on error
            this.fetchPendingOrders();
          });
      } else {
        console.log(`Order #${orderId} not found in current orders, refreshing all orders`);
        // If the order isn't found in the current array, refresh all orders
        this.fetchPendingOrders();
      }
    },

    // Add direct DOM manipulation method to fix pending approval status when reactivity fails
    forceFixPendingUI(orderId) {
      console.log(`CRITICAL FIX: Using direct DOM manipulation to fix UI for order #${orderId}`);
      
      try {
        // Find the order element
        const orderElement = document.querySelector(`.order-item[data-order-id="${orderId}"]`);
        if (!orderElement) {
          console.log("Order element not found for direct fix");
          return false;
        }
        
        // Remove the critical class that causes the red border
        orderElement.classList.remove('order-pending-approval');
        
        // Add a success indicator class
        orderElement.classList.add('order-approved-success');
        
        // Find and update status text
        const statusParagraphs = orderElement.querySelectorAll('p');
        for (const paragraph of statusParagraphs) {
          if (paragraph.textContent.includes("Status:")) {
            paragraph.innerHTML = '<strong>Status:</strong> pending';
            console.log('Updated status text');
          }
        }
        
        // Remove the "PENDING APPROVAL" label
        // Create a specific style for this order element
        const styleId = `order-${orderId}-fix`;
        let styleElement = document.getElementById(styleId);
        
        if (!styleElement) {
          styleElement = document.createElement('style');
          styleElement.id = styleId;
          document.head.appendChild(styleElement);
        }
        
        styleElement.textContent = `
          .order-item[data-order-id="${orderId}"]::before {
            display: none !important;
          }
          .order-item[data-order-id="${orderId}"].order-pending-approval {
            background-color: #ffffff !important;
            border: 2px solid #ddd !important;
            box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1) !important;
          }
          .order-item[data-order-id="${orderId}"] h3 {
            color: #333 !important;
          }
          .order-item[data-order-id="${orderId}"] .order-total {
            color: #333 !important;
          }
          .order-approved-success {
            animation: successPulse 1s;
          }
          @keyframes successPulse {
            0% { background-color: #e6ffe6; }
            50% { background-color: #ccffcc; }
            100% { background-color: #ffffff; }
          }
        `;
        
        console.log("Direct DOM manipulation complete - critical CSS override applied");
        return true;
      } catch (error) {
        console.error("Error during direct DOM manipulation:", error);
        return false;
      }
    },

    // Completely bypass Vue for this critical update
    handleApprovalBypassVue(orderId) {
      // Force a direct manipulation of the DOM 
      const fixed = this.forceFixPendingUI(orderId);
      
      // If direct manipulation succeeded, set a flag to prevent further attempts
      if (fixed) {
        // Store fixed orders in localStorage to avoid repeated fixes
        const fixedOrders = JSON.parse(localStorage.getItem('fixedApprovedOrders') || '[]');
        if (!fixedOrders.includes(orderId)) {
          fixedOrders.push(orderId);
          localStorage.setItem('fixedApprovedOrders', JSON.stringify(fixedOrders));
        }
        
        // Show feedback to admin
        this.notificationMessage = `The customer has APPROVED the order adjustments. Order ID: ${orderId}`;
        this.notificationClass = "open-notification";
        this.showNotification();
      }
      
      // Force refresh orders after a short delay
      setTimeout(() => {
        this.fetchPendingOrders();
      }, 1000);
    },

    toggleChangePassword() {
      this.showChangePasswordModal = !this.showChangePasswordModal;
      if (this.isSidebarOpen && window.innerWidth <= 768) {
        this.toggleSidebar();
      }
    },

    updatePassword() {
      // Check if current password is correct
      const currentAdminPassword = localStorage.getItem('adminPassword') || 'admin123';
      
      // Reset message
      this.passwordMessage = '';
      this.passwordMessageType = '';
      
      // Validate current password
      if (this.passwordData.currentPassword !== currentAdminPassword) {
        this.passwordMessage = 'Current password is incorrect';
        this.passwordMessageType = 'error';
        return;
      }
      
      // Validate new password
      if (this.passwordData.newPassword.length < 6) {
        this.passwordMessage = 'New password must be at least 6 characters';
        this.passwordMessageType = 'error';
        return;
      }
      
      // Validate password confirmation
      if (this.passwordData.newPassword !== this.passwordData.confirmPassword) {
        this.passwordMessage = 'New passwords do not match';
        this.passwordMessageType = 'error';
        return;
      }
      
      // Update the password in localStorage
      localStorage.setItem('adminPassword', this.passwordData.newPassword);
      
      // Show success message
      this.passwordMessage = 'Password updated successfully!';
      this.passwordMessageType = 'success';
      
      // Clear form after a delay
      setTimeout(() => {
        this.passwordData = {
          currentPassword: '',
          newPassword: '',
          confirmPassword: ''
        };
        
        // Close the modal after 2 seconds
        setTimeout(() => {
          this.toggleChangePassword();
        }, 1000);
      }, 1000);
    }
  },

  mounted() {
    this.fetchPendingOrders();
    // Set up automatic refresh every 60 seconds
    this.refreshInterval = setInterval(() => {
      this.fetchPendingOrders();
    }, 60000);
    
    // Check localStorage for cafe status
    const storedCafeStatus = localStorage.getItem('isCafeOpen');
    if (storedCafeStatus !== null) {
      this.isCafeOpen = storedCafeStatus === 'true';
    }
    
    // Load orderReadyStatus from localStorage if available
    const savedOrderReadyStatus = localStorage.getItem('orderReadyStatus');
    if (savedOrderReadyStatus) {
      this.orderReadyStatus = JSON.parse(savedOrderReadyStatus);
    }
    
    // Event listener for customer approval/decline responses
    window.addEventListener('storage', this.handleStorageEvent);
    
    // Connect to WebSocket
    this.connectWebSocket();
    
    // Add a global document event listener for a custom event we'll dispatch on customer approval
    document.addEventListener('customer-approval', (event) => {
      if (event.detail && event.detail.orderId) {
        console.log('Got customer-approval event at document level:', event.detail);
        this.handleApprovalBypassVue(event.detail.orderId);
      }
    });
    
    // Dispatch a custom approval event if we have any approvals stored in localStorage
    // This helps sync UI state on page load
    const fixedOrders = JSON.parse(localStorage.getItem('fixedApprovedOrders') || '[]');
    fixedOrders.forEach(orderId => {
      setTimeout(() => {
        const orderElement = document.querySelector(`.order-item[data-order-id="${orderId}"].order-pending-approval`);
        if (orderElement) {
          console.log(`Found previously fixed order #${orderId} still showing pending, re-fixing`);
          this.forceFixPendingUI(orderId);
        }
      }, 1000);
    });
  },

  beforeUnmount() {
    if (this.refreshInterval) {
      clearInterval(this.refreshInterval);
    }
    
    if (this.pingInterval) {
      clearInterval(this.pingInterval);
    }
    
    if (this.ws) {
      this.ws.close();
    }
    
    // Remove event listeners
    window.removeEventListener('storage', this.handleStorageEvent);
    document.removeEventListener('customer-approval', this.handleApprovalBypassVue);
  }
};
</script>

<style scoped>
/* Base styles for all screen sizes */
.notifications-page {
  height: 100vh;
  display: flex;
  background-color: #ffffff;
}

/* Top Bar Styles to match dashboard */
.top-bar {
  display: flex;
  align-items: center;
  background-image: linear-gradient(to right, #E54F70, #ed9598);
  padding: 0 15px;
  height: 60px;
  width: 100%;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 100;
}

.content.shifted .top-bar {
  left: 300px; /* Adjust left position when sidebar is open */
  width: calc(100% - 300px); /* Adjust width when sidebar is open */
}

.content-below-top-bar {
  margin-top: 70px; /* Add margin to account for the fixed top bar */
  padding: 10px 20px;
}

.centered-content {
  display: flex;
  align-items: center;
  justify-content: flex-start;
  padding: 0 0 0 5px; /* Reduce left padding to move button closer to edge */
  height: 100%;
}

.logo-container {
  display: flex;
  align-items: center;
}

.cafe-title {
  color: white;
  font-weight: bold;
  font-size: 20px;
  white-space: nowrap;
}

/* Menu Button */
.menu-button {
  position: fixed;
  top: 15px;
  left: 15px;
  z-index: 300;
  background: #d12f7a;
  color: white;
  padding: 12px 15px;
  font-size: 20px;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s ease-in-out;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  display: none; /* Hide the floating button */
}

.menu-button:hover {
  background: #b82d67;
  transform: translateY(-2px);
  box-shadow: 0 6px 8px rgba(0, 0, 0, 0.15);
}

.menu-icon-container {
  position: relative;
  display: inline-block;
  font-size: 24px;
}

/* Sidebar */
.sidebar {
  position: fixed;
  top: 0;
  left: -300px;
  height: 100vh;
  width: 300px;
  background: #f5f5f5;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  z-index: 1000;
  display: flex;
  flex-direction: column;
  padding: 20px 0;
  box-shadow: 4px 0 15px rgba(0, 0, 0, 0.1);
  color: #333;
}

.sidebar.open {
  left: 0;
}

.close-sidebar {
  position: absolute;
  top: 15px;
  right: 15px;
  background: rgba(209, 47, 122, 0.1);
  border: none;
  font-size: 20px;
  cursor: pointer;
  color: #d12f7a;
  padding: 8px 12px;
  border-radius: 50%;
  transition: all 0.3s ease;
}

.close-sidebar:hover {
  background: rgba(209, 47, 122, 0.2);
  transform: rotate(90deg);
}

/* User Profile Section */
.user-profile-section {
  padding: 30px 20px;
  text-align: center;
  margin-bottom: 20px;
  background: rgba(209, 47, 122, 0.1);
  border-radius: 15px;
  margin: 0 15px 20px;
}

.user-profile-section h3 {
  color: #d12f7a;
  margin: 0;
  font-size: 28px;
  font-weight: 600;
  text-shadow: none;
}

/* Utility Section */
.utility-section {
  padding: 15px;
  margin: 0 15px;
  background: white;
  border-radius: 15px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.utility-button {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 20px;
  background: transparent;
  border: none;
  cursor: pointer;
  color: #333;
  font-size: 16px;
  text-decoration: none;
  transition: all 0.3s ease;
  border-radius: 10px;
  margin-bottom: 8px;
  position: relative;
  overflow: hidden;
}

.utility-button:last-child {
  margin-bottom: 0;
}

.utility-button:hover {
  background: rgba(209, 47, 122, 0.1);
  transform: translateX(5px);
}

.utility-button i {
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #d12f7a;
  border-radius: 8px;
  font-size: 16px;
  color: white;
  transition: all 0.3s ease;
}

.utility-button:hover i {
  transform: rotate(10deg);
}

.utility-button span {
  font-weight: 500;
}

.utility-button.logout {
  background: rgba(244, 67, 54, 0.1);
  color: #f44336;
  margin-top: 8px;
  border-top: 1px solid rgba(0, 0, 0, 0.05);
}

.utility-button.logout i {
  background: #f44336;
}

.utility-button.logout:hover {
  background: rgba(244, 67, 54, 0.15);
}

/* Cafe Status Section */
.cafe-status-section {
  padding: 20px;
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

.cafe-toggle-btn {
  font-size: 16px;
  padding: 12px 24px;
  border: 2px solid #d12f7a;
  cursor: pointer;
  border-radius: 30px;
  transition: all 0.3s ease;
  font-weight: 600;
  width: 100%;
  background: white;
  color: #d12f7a;
  text-transform: uppercase;
  letter-spacing: 1px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.cafe-toggle-btn::before {
  content: '🏪';
  font-size: 20px;
}

.cafe-toggle-btn:hover {
  background: rgba(209, 47, 122, 0.1);
  transform: translateY(-2px);
}

.cafe-toggle-btn.open-btn {
  border-color: #4CAF50;
  color: #4CAF50;
}

.cafe-toggle-btn.open-btn::before {
  content: '✅';
}

.cafe-toggle-btn.closed-btn {
  border-color: #f44336;
  color: #f44336;
}

.cafe-toggle-btn.closed-btn::before {
  content: '🚫';
}

/* Utility Divider */
.utility-divider {
  border: none;
  height: 1px;
  background: rgba(0, 0, 0, 0.1);
  margin: 15px;
  border-radius: 1px;
}

/* Main Content Area */
.content {
  flex: 1;
  margin-left: 0;
  padding: 20px;
  transition: margin-left 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.content.shifted {
  margin-left: 300px;
}

/* Overlay */
.overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(4px);
  z-index: 999;
  opacity: 0;
  transition: opacity 0.3s ease;
}

.overlay.visible {
  opacity: 1;
}

/* Search Bar */
.search-bar {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 20px auto;
  width: 90%;
  max-width: 400px;
  background-color: transparent;
  border-radius: 30px;
  border: 2px solid #d12f7a;
}

.search-input {
  width: 100%;
  padding: 10px 15px;
  font-size: 16px;
  border: none;
  outline: none;
  background-color: transparent;
  color: #333;
  border-radius: 30px;
}

/* Orders Container */
.orders-container {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
}

.orders-container h2 {
  text-align: center;
  color: #d12f7a;
  margin-bottom: 20px;
}

.orders-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 20px;
  width: 100%;
}

/* Order Item */
.order-item {
  background-color: #f8d2e4;
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  height: auto;
  transition: all 0.3s ease-in-out;
}

.order-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 8px rgba(0, 0, 0, 0.15);
}

.order-details h3 {
  color: #d12f7a;
  margin-top: 0;
}

.order-details p {
  margin: 5px 0;
}

.items-section {
  margin-top: 15px; 
  flex-grow: 1; /* Allow the items section to expand and adapt */
}

.order-actions {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-top: 15px;
}

ul {
  list-style-type: none;
  padding-left: 20px;
  margin: 0;
}

.order-total {
  margin-top: 10px;
  font-weight: bold;
  color: #d12f7a;
  text-align: center; /* Ensure it aligns nicely with the rest of the content */
}

/* Button styles */
button.mark-completed-btn {
  background-color: #d12f7a;
  color: white;
  border: none;
  padding: 8px 10px;
  border-radius: 5px;
  cursor: pointer;
  width: 100%;
  transition: all 0.3s ease;
}

button.mark-completed-btn:disabled,
button.mark-completed-btn.disabled {
  background-color: #cccccc;
  cursor: not-allowed;
  opacity: 0.7;
}

button.mark-completed-btn:not(:disabled):hover {
  background-color: #b82d67;
}

.order-ready-btn {
  background-color: #4caf50; /* Green background for success */
  color: white; /* White text */
  padding: 8px 10px;
  font-size: 14px;
  border: none;
  border-radius: 5px; /* Rounded corners */
  cursor: pointer;
  width: 100%;
  transition: background-color 0.3s ease, transform 0.3s ease;
}

.order-ready-btn:hover {
  background-color: #45a049; /* Slightly darker green when hovered */
  transform: translateY(-2px); /* Slight upward movement on hover */
}

.order-ready-btn:active {
  background-color: #388e3c; /* Even darker green when clicked */
  transform: translateY(0); /* Reset the movement after click */
}

button.decline-btn {
  background-color: #f5a5a5;
  color: white;
  border: none;
  padding: 8px 10px;
  border-radius: 5px;
  cursor: pointer;
  width: 100%;
}

button.decline-btn:hover {
  background-color: #f17b7b;
}

/* Notification styles */
.notification-popup,
.notification-sent-popup {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  padding: 15px 30px;
  border-radius: 8px;
  box-shadow: 0 6px 8px rgba(0, 0, 0, 0.2);
  font-size: 18px;
  min-width: 200px;
  max-width: 90%;
  z-index: 1000;
  transition: opacity 0.3s ease, transform 0.3s ease;
  opacity: 1;
  text-align: center;
}

.notification-sent-popup {
  background-color: rgb(82, 13, 45);
  color: white;
}

.notification-popup.hide {
  opacity: 0;
  transform: translateX(-50%) translateY(-20px);
}

.notification-popup button,
.notification-sent-popup button {
  background-color: #fff;
  color: #007bff;
  border: none;
  padding: 5px 10px;
  border-radius: 3px;
  cursor: pointer;
  margin-left: 10px;
  font-size: 14px;
}

.notification-popup button:hover,
.notification-sent-popup button:hover {
  background-color: #f1f1f1;
}

.notification-popup.open-notification {
  background-color: #4caf50;
  color: white;
}

.notification-popup.closed-notification {
  background-color: red;
  color: white;
}

.highlighted-order-details {
  display: inline-block;
  background-color: #f8d2e4;
  color: #d12f7a;
  padding: 10px;
  border-radius: 5px;
  margin-top: 10px;
  font-weight: bold;
  border: 2px solid #d12f7a;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  width: 100%;
  text-align: center;
  box-sizing: border-box;
}

.highlighted-order-details::before {
  content: "——— ";
  font-size: 24px;
  font-weight: bold;
  color: #d12f7a;
}

.loading {
  text-align: center;
  color: #d12f7a;
  font-size: 20px;
  margin-top: 20px;
}

.no-orders {
  text-align: center;
  margin: 30px 0;
}

.highlighted-time {
  color: #d12f7a;
  font-weight: bold;
}

.dark-mode .highlighted-time {
  color: #f8d2e4;
}

.section-divider {
  border-top: 1px solid #ddd;
  margin: 30px 0;
}

/* Media Queries for Responsive Design */
@media (max-width: 768px) {
  .header {
    flex-direction: column;
    align-items: center;
    gap: 15px;
  }
  
  .header-buttons {
    order: 3;
    width: 100%;
    justify-content: center;
  }
  
  h1 {
    order: 1;
    width: 100%;
  }
  
  .order-record-button {
    order: 2;
    width: 100%;
    max-width: 200px;
  }
  
  .logout-button {
    order: 4;
    width: 100%;
    max-width: 200px;
  }
  
  .orders-list {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 480px) {
  .notifications-page {
    padding: 10px;
  }
  
  .cafe-toggle-btn {
    width: 180px;
    font-size: 14px;
  }
  
  .notification-popup,
  .notification-sent-popup {
    font-size: 16px;
    padding: 10px 20px;
  }
  
  .order-item {
    padding: 12px;
  }
  
  .decline-container {
    padding: 10px;
  }
  
  .decline-buttons {
    flex-direction: column;
  }
  
  .decline-submit-btn,
  .decline-cancel-btn {
    width: 100%;
  }
}

/* Menu Editor Modal Styles */
.menu-editor-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1001;
  overflow-y: auto;
}

.menu-editor-content {
  background-color: white;
  width: 90%;
  max-width: 800px;
  max-height: 90vh;
  border-radius: 10px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.menu-editor-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  background-color: #f8d2e4;
  border-bottom: 1px solid #e0e0e0;
}

.menu-editor-header h2 {
  color: #d12f7a;
  margin: 0;
}

.close-modal-btn {
  background: none;
  border: none;
  color: #d12f7a;
  font-size: 20px;
  cursor: pointer;
  padding: 5px;
}

.close-modal-btn:hover {
  color: #b82d67;
}

.menu-editor-body {
  padding: 20px;
  overflow-y: auto;
  max-height: calc(90vh - 70px); /* Subtract header height */
}

/* Media query adjustments for the modal */
@media (max-width: 768px) {
  .menu-editor-content {
    width: 95%;
    max-height: 95vh;
  }
}

@media (max-width: 480px) {
  .menu-editor-header {
    padding: 10px 15px;
  }
  
  .menu-editor-body {
    padding: 15px;
  }
}

/* Completion Confirmation Popup Styles */
.completion-confirmation-popup {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1002;
}

.completion-confirmation-content {
  background-color: white;
  padding: 25px;
  border-radius: 10px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  text-align: center;
  max-width: 400px;
  width: 90%;
}

.completion-confirmation-content h3 {
  color: #d12f7a;
  margin-top: 0;
  margin-bottom: 15px;
  font-size: 1.5em;
}

.completion-confirmation-content p {
  margin-bottom: 20px;
  font-size: 1.1em;
  color: #333;
}

.confirmation-buttons {
  display: flex;
  justify-content: center;
  gap: 15px;
}

.confirm-yes-btn,
.confirm-no-btn {
  padding: 10px 25px;
  border: none;
  border-radius: 5px;
  font-size: 1.1em;
  cursor: pointer;
  transition: all 0.3s ease;
}

.confirm-yes-btn {
  background-color: #d12f7a;
  color: white;
}

.confirm-yes-btn:hover {
  background-color: #b82d67;
  transform: translateY(-2px);
}

.confirm-no-btn {
  background-color: #f5a5a5;
  color: white;
}

.confirm-no-btn:hover {
  background-color: #f17b7b;
  transform: translateY(-2px);
}

/* Media query adjustments for the confirmation popup */
@media (max-width: 480px) {
  .completion-confirmation-content {
    padding: 20px;
    width: 85%;
  }

  .confirmation-buttons {
    flex-direction: column;
    gap: 10px;
  }

  .confirm-yes-btn,
  .confirm-no-btn {
    width: 100%;
    padding: 12px;
  }
}

/* Stock Management Modal Styles */
.stock-manager-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1001;
}

.stock-manager-content {
  background-color: white;
  width: 90%;
  max-width: 1000px;
  max-height: 90vh;
  border-radius: 10px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
}

.stock-manager-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  background-color: #f8d2e4;
  border-top-left-radius: 10px;
  border-top-right-radius: 10px;
}

.stock-manager-header h2 {
  color: #d12f7a;
  margin: 0;
}

.stock-manager-body {
  padding: 20px;
  overflow-y: auto;
}

.stock-search-bar {
  margin-bottom: 20px;
}

.stock-table-container {
  overflow-x: auto;
}

.stock-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 20px;
}

.stock-table th,
.stock-table td {
  padding: 12px;
  text-align: left;
  border-bottom: 1px solid #ddd;
}

.stock-table th {
  background-color: #f8d2e4;
  color: #d12f7a;
}

.status-out {
  color: #f44336;
  font-weight: bold;
}

.status-low {
  color: #ff9800;
  font-weight: bold;
}

.status-ok {
  color: #4caf50;
  font-weight: bold;
}

.status-disabled {
  color: #9e9e9e;
  font-weight: bold;
}

.status-enabled {
  color: #2196f3;
  font-weight: bold;
}

.action-buttons {
  display: flex;
  gap: 10px;
}

.update-stock-btn {
  background-color: #d12f7a;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 20px;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.3s ease;
  box-shadow: 0 2px 4px rgba(209, 47, 122, 0.2);
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.update-stock-btn::before {
  content: '📦';
  font-size: 16px;
}

.update-stock-btn:hover {
  background-color: #b82d67;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(209, 47, 122, 0.3);
}

.update-stock-btn:active {
  transform: translateY(0);
  box-shadow: 0 2px 4px rgba(209, 47, 122, 0.2);
}

.stock-update-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1002;
}

.stock-update-content {
  background-color: white;
  padding: 20px;
  border-radius: 10px;
  width: 90%;
  max-width: 400px;
}

.stock-update-form {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.form-group label {
  font-weight: bold;
}

.form-group input,
.form-group select {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.update-buttons {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
  margin-top: 20px;
}

.confirm-btn,
.cancel-btn {
  padding: 8px 16px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.confirm-btn {
  background-color: #d12f7a;
  color: white;
}

.cancel-btn {
  background-color: #f5a5a5;
  color: white;
}

.low-stock {
  color: #ff9800;
  font-weight: bold;
}

.out-of-stock {
  color: #f44336;
  font-weight: bold;
}

.stock-filters {
  display: flex;
  gap: 10px;
  margin-bottom: 15px;
  width: 100%;
}

.category-filter {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  min-width: 150px;
  background-color: white;
  cursor: pointer;
}

.category-filter:hover {
  border-color: #d12f7a;
}

.dark-mode .category-filter {
  background-color: #333;
  color: white;
  border-color: #555;
}

.dark-mode .category-filter:hover {
  border-color: #f8c6d0;
}

/* Add styles for the new header menu button */
.menu-button-header {
  background: #d12f7a;
  color: white;
  padding: 8px 12px;
  font-size: 18px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease-in-out;
  margin-right: 15px;
  margin-left: 0; /* Ensure no left margin */
  display: flex;
  align-items: center;
  justify-content: center;
}

.menu-button-header:hover {
  background: #b82d67;
}

/* New styles for decline modal and order highlighting */
.order-declined-state {
  background-color: #ffe6e6 !important;
  border: 2px solid #ff0000 !important;
}

.order-pending-approval {
  background-color: #ffe6e6 !important;
  border: 2px solid #ff0000 !important;
  box-shadow: 0 4px 10px rgba(255, 0, 0, 0.2) !important;
  position: relative;
  transition: all 0.5s ease-in-out;
}

.order-pending-approval::before {
  content: 'PENDING APPROVAL';
  position: absolute;
  top: 0;
  right: 0;
  background-color: #ff0000;
  color: white;
  font-size: 12px;
  font-weight: bold;
  padding: 4px 8px;
  border-bottom-left-radius: 8px;
}

.order-pending-approval h3 {
  color: #ff0000 !important;
}

.order-pending-approval .order-details p {
  color: #333 !important;
}

.order-pending-approval .order-total {
  color: #ff0000 !important;
  font-weight: bold;
}

.decline-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.decline-modal-content {
  background-color: white;
  border-radius: 8px;
  width: 90%;
  max-width: 600px;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.25);
}

.decline-modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  border-bottom: 1px solid #eee;
  background-color: #f8d2e4;
}

.decline-modal-header h3 {
  margin: 0;
  color: #d12f7a;
}

.decline-modal-body {
  padding: 20px;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: bold;
  color: #333;
}

.form-group textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  resize: vertical;
}

.items-adjustment {
  max-height: 200px;
  overflow-y: auto;
  border: 1px solid #eee;
  border-radius: 4px;
  margin-bottom: 10px;
}

.item-adjust-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  border-bottom: 1px solid #eee;
}

.item-adjust-row:last-child {
  border-bottom: none;
}

.item-name {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.quantity-controls {
  display: flex;
  align-items: center;
  margin: 0 10px;
}

.quantity-controls button {
  width: 30px;
  height: 30px;
  border: 1px solid #ddd;
  background-color: #f0f0f0;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
}

.quantity-controls button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.quantity-controls input {
  width: 50px;
  text-align: center;
  margin: 0 5px;
  padding: 5px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.item-price {
  min-width: 80px;
  text-align: right;
  font-weight: bold;
}

.adjusted-total {
  text-align: right;
  font-size: 16px;
  margin-top: 10px;
  color: #d12f7a;
}

.decline-modal-actions {
  display: flex;
  justify-content: space-between;
  margin-top: 20px;
}

.send-approval-btn {
  background-color: #4caf50;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}

.confirm-decline-btn {
  background-color: #f44336;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}

/* Animation for order approval transition */
@keyframes orderApproved {
  0% {
    background-color: #ffe6e6;
    border-color: #ff0000;
    box-shadow: 0 4px 10px rgba(255, 0, 0, 0.2);
  }
  50% {
    background-color: #e6ffe6;
    border-color: #4caf50;
    box-shadow: 0 4px 10px rgba(76, 175, 80, 0.3);
  }
  100% {
    background-color: white;
    border-color: #ddd;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  }
}

.order-approved-animation {
  animation: orderApproved 1.5s ease-in-out forwards;
}

/* Add the success animation styles */
@keyframes successPulse {
  0% { background-color: #fff; }
  25% { background-color: #e6ffe6; }
  50% { background-color: #ccffcc; }
  75% { background-color: #e6ffe6; }
  100% { background-color: #fff; }
}

.order-approved-success {
  animation: successPulse 2s ease;
  border: 2px solid #4CAF50 !important;
  box-shadow: 0 4px 8px rgba(76, 175, 80, 0.3) !important;
}

/* Change Password Modal Styles */
.password-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1003;
  overflow-y: auto;
}

.password-modal-content {
  background-color: white;
  width: 90%;
  max-width: 800px;
  max-height: 90vh;
  border-radius: 10px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.password-modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  background-color: #f8d2e4;
  border-bottom: 1px solid #e0e0e0;
}

.password-modal-header h2 {
  color: #d12f7a;
  margin: 0;
}

.close-modal-btn {
  background: none;
  border: none;
  color: #d12f7a;
  font-size: 20px;
  cursor: pointer;
  padding: 5px;
}

.close-modal-btn:hover {
  color: #b82d67;
}

.password-modal-body {
  padding: 20px;
  overflow-y: auto;
  max-height: calc(90vh - 70px); /* Subtract header height */
}

.password-form {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.form-group label {
  font-weight: bold;
}

.form-group input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.form-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 20px;
}

.save-password-btn {
  background-color: #d12f7a;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 5px;
  cursor: pointer;
}

.password-message {
  color: #ff0000;
  font-size: 14px;
  margin-top: 5px;
}

.password-message.success {
  color: #4CAF50;
}

.password-message.error {
  color: #f44336;
}

</style>