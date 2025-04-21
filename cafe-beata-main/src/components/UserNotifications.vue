<template>
  <div :class="['notifications-container', { 'dark-mode': isDarkMode }]">
    <!-- Floating Notification Button -->
    <div class="floating-notification" @click="toggleNotificationPanel">
      <i class="fas fa-bell"></i>
      <span v-if="unreadCount > 0" class="floating-notification-badge">{{ unreadCount }}</span>
    </div>

    <!-- Notification Panel (Modal) -->
    <div v-if="isPanelOpen" class="notification-panel-overlay" @click="closePanel">
      <div class="notifications-card" @click.stop>
        <div class="notifications-header">
          <h2>Notifications</h2>
          <div class="notification-controls">
            <button class="close-panel" @click="closePanel">✕</button>
          </div>
        </div>

        <!-- Notifications List -->
        <div class="notifications-list">
          <!-- Check if there are no notifications -->
          <p v-if="filteredNotifications.length === 0" class="no-notifications">
            No notifications at the moment.
          </p>

          <!-- Render notifications if there are any -->
          <div v-for="(notification, index) in filteredNotifications" :key="index" class="notification-item">
            <div class="notification-avatar">
              <img :src="getAvatarImage()" alt="Café Beata avatar">
            </div>
            <div class="notification-content">
              <div class="notification-header">
                <span class="notification-name">Café Beata</span>
                <span class="notification-time">{{ formatTimeAgo(notification.timestamp) }}</span>
              </div>
              <p class="notification-message" v-html="formatNotificationMessage(notification.message)"></p>
              <p class="order-details" v-if="extractOrderDetails(notification.message)">
                {{ extractOrderDetails(notification.message) }}
              </p>
              
              <!-- Approval Buttons for notifications that require approval -->
              <div v-if="notification.requiresApproval" class="approval-actions">
                <button @click="approveAdjustment(notification, index)" class="approve-btn">
                  Approve
                </button>
                <button @click="declineAdjustment(notification, index)" class="decline-btn">
                  Decline
                </button>
              </div>
            </div>
          </div>
        </div>

        <div class="notifications-footer">
          <div class="action-buttons">
            <button class="mark-read-btn" @click="showMarkReadConfirmation = true">Mark All as Read</button>
            <button class="clear-btn" @click="clearNotifications">Clear Notifications</button>
          </div>
        </div>
      </div>
    </div>

    <!-- Mark As Read Confirmation Popup -->
    <div v-if="showMarkReadConfirmation" class="confirmation-popup">
      <div class="confirmation-content">
        <h3>Confirm Action</h3>
        <p>Are you sure you want to mark all notifications as read?</p>
        <div class="confirmation-buttons">
          <button @click="markAllAsRead" class="confirm-yes-btn">Yes</button>
          <button @click="showMarkReadConfirmation = false" class="confirm-no-btn">No</button>
        </div>
      </div>
    </div>

    <!-- New styles for loading and success messages -->
    <div v-if="showLoadingMessage" class="loading-message">
      {{ loadingMessage }}
    </div>

    <div v-if="showSuccessMessage" class="success-message-floating">
      {{ successMessage }}
    </div>
  </div>
</template>

<script>
import { eventBus } from "@/utils/eventBus"; // Correct the path if needed

export default {
  data() {
    return {
      notifications: [],
      ws: null,
      wsConnected: false,
      wsReconnectAttempts: 0,
      maxReconnectAttempts: 5,
      reconnectInterval: 3000,
      isDarkMode: localStorage.getItem("darkMode") === "true",
      showMarkReadConfirmation: false,
      isPanelOpen: false,
      unreadCount: 0,
      showLoadingMessage: false,
      loadingMessage: '',
      showSuccessMessage: false,
      successMessage: ''
    };
  },
  computed: {
    filteredNotifications() {
      // Only show notifications from Café Beata, not from other sources like "Lors"
      return this.notifications.map(notif => {
        // Always set the customerName to Café Beata regardless of original source
        return {
          ...notif,
          customerName: 'Café Beata'
        };
      });
    }
  },
  methods: {
    toggleNotificationPanel() {
      this.isPanelOpen = !this.isPanelOpen;
      if (this.isPanelOpen) {
        document.body.style.overflow = 'hidden';
      } else {
        document.body.style.overflow = '';
      }
    },
    
    closePanel() {
      this.isPanelOpen = false;
      document.body.style.overflow = '';
    },
    
    getAvatarImage() {
      // Always use Café Beata for the avatar
      return `https://ui-avatars.com/api/?name=Café+Beata&background=E54F70&color=fff&size=40`;
    },
    
    // Format notification message to be more user-friendly
    formatNotificationMessage(message) {
      // Check if this is an order adjustment message
      if (message.includes("adjust your order") || message.includes("review the adjusted order")) {
        // For adjustments, extract and format the specific adjustment reasons
        if (message.includes("The following adjustments were made:")) {
          const beforeAdjustmentDetails = message.split("The following adjustments were made:")[0].trim();
          const adjustmentDetails = message.split("The following adjustments were made:")[1].split("Please review")[0].trim();
          
          // Return a nicely formatted message
          return `
            <strong>${beforeAdjustmentDetails}</strong>
            <div class="adjustment-details">
              ${adjustmentDetails}
            </div>
          `;
        } else {
          // If no specific adjustments are listed, just show the main message
          return message.split("Please review the adjusted order:")[0].trim();
        }
      }
      
      // Normal message formatting
      return message.replace(/(Order details:.*?Total: ₱\d+(\.\d{2})?)/, '')
                    .replace(/Your order/, 'Your order')
                    .replace(/\s{2,}/g, ' ') // Remove extra spaces
                    .trim();
    },
    
    // Extract just the order details to display in a separate element
    extractOrderDetails(message) {
      const orderDetailsMatch = message.match(/(Order details:(.*?)Total: ₱\d+(\.\d{2})?)/);
      return orderDetailsMatch ? orderDetailsMatch[0] : '';
    },
    
    formatTimeAgo(timestamp) {
      const now = new Date();
      const diff = now - new Date(timestamp);
      const minutes = Math.floor(diff / 1000 / 60);
      const hours = Math.floor(diff / 1000 / 60 / 60);
      const days = Math.floor(diff / 1000 / 60 / 60 / 24);

      if (days > 0) return `${days} day${days > 1 ? "s" : ""} ago`;
      if (hours > 0) return `${hours} hour${hours > 1 ? "s" : ""} ago`;
      if (minutes > 0) return `${minutes} minute${minutes > 1 ? "s" : ""} ago`;
      return "Just now";
    },

    fetchNotifications() {
      const userName = localStorage.getItem("userName"); 
      if (userName) {
        const userNotificationsKey = `user_notifications_${userName}`;
        let storedNotifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
        
        // Remove duplicate notifications (same order ID and same message)
        const uniqueNotifications = [];
        const seen = new Set();
        
        storedNotifications.forEach(notification => {
          // Create a unique key using orderId and message
          const uniqueKey = `${notification.orderId}-${notification.message}`;
          
          if (!seen.has(uniqueKey)) {
            seen.add(uniqueKey);
            // Ensure read property is defined
            if (notification.read === undefined) {
              notification.read = false;
            }
            uniqueNotifications.push(notification);
          }
        });
        
        // Sort notifications by timestamp (newest first)
        uniqueNotifications.sort((a, b) => {
          const dateA = new Date(a.timestamp);
          const dateB = new Date(b.timestamp);
          return dateB - dateA;
        });
        
        // Save the deduplicated notifications back to localStorage
        localStorage.setItem(userNotificationsKey, JSON.stringify(uniqueNotifications));
        
        // Update the notifications array with the deduplicated list
        this.notifications = uniqueNotifications;
        
        // Force update the unread count and update DOM
        const unreadCount = this.notifications.filter(notification => !notification.read).length;
        this.unreadCount = unreadCount;
        eventBus.notificationsCount = unreadCount;
        localStorage.setItem("unread_notifications", unreadCount);
        
        // Log for debugging
        console.log(`Fetched ${this.notifications.length} notifications, ${unreadCount} unread`);
        
        // If there are unread notifications and we just connected, show alert
        if (this.wsConnected && unreadCount > 0) {
          console.log(`Found ${unreadCount} unread notifications on connection`);
        }
        
        // Explicitly dispatch notification update event
        window.dispatchEvent(new CustomEvent("notificationUpdated"));
        
        return this.notifications;
      } else {
        this.notifications = [];
        // Reset notification count
        this.unreadCount = 0;
        eventBus.notificationsCount = 0;
        localStorage.setItem("unread_notifications", 0);
        return [];
      }
    },

    addNewNotification(notification) {
      const userName = localStorage.getItem("userName");
      if (!userName) return false;
      
      const userNotificationsKey = `user_notifications_${userName}`;
      let notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
      
      // Always ensure the notification has the current timestamp if not provided
      if (!notification.timestamp) {
        notification.timestamp = new Date().toISOString();
      }
      
      // Check if a notification with the same order ID and message already exists
      const existingIndex = notifications.findIndex(n => 
        n.orderId === notification.orderId && n.message === notification.message
      );
      
      // Only add the notification if it doesn't already exist
      if (existingIndex === -1) {
        // Make sure read status is explicitly set to false for new notifications
        notification.read = false;
        
        // Add notification at the beginning of the array (newest first)
        notifications.unshift(notification);
        
        // Sort notifications by timestamp (newest first)
        notifications.sort((a, b) => {
          const dateA = new Date(a.timestamp);
          const dateB = new Date(b.timestamp);
          return dateB - dateA;
        });
        
        // Save the updated notifications to localStorage
        localStorage.setItem(userNotificationsKey, JSON.stringify(notifications));

        // Update the notifications in the component
        this.notifications = notifications;
        
        // Directly update the component state for immediate UI update
        this.unreadCount = this.notifications.filter(n => !n.read).length;
        
        // Update eventBus for other components
        eventBus.notificationsCount = this.unreadCount;
        
        // Update localStorage for persistence
        localStorage.setItem("unread_notifications", this.unreadCount);
        
        // Show notification alert
        this.showNewNotificationAlert();
        
        // Log for debugging
        console.log(`New notification added for ${userName}, badge count: ${this.unreadCount}`);
        
        // Dispatch event to notify other components
        window.dispatchEvent(new CustomEvent("notificationUpdated"));
        
        return true; // Notification was added
      }
      
      return false; // Notification already exists
    },

    // Calculate estimated preparation time based on order items
    calculateEstimatedTime(items) {
      // Check if the order contains only drinks, only food, or both
      const hasDrinks = items.some(item => 
        item.category && 
        (item.category.toLowerCase().includes('drink') || 
         item.category === 'Juice Drinks' || 
         item.category === 'Chocolate Drinks' ||
         item.category === 'Coffee')
      );
      
      const hasFood = items.some(item => 
        item.category && 
        !item.category.toLowerCase().includes('drink') && 
        item.category !== 'Juice Drinks' && 
        item.category !== 'Chocolate Drinks' &&
        item.category !== 'Coffee'
      );
      
      // Return the appropriate estimated time
      if (hasDrinks && !hasFood) {
        return "10-12 minutes";
      } else if (hasFood || (hasDrinks && hasFood)) {
        return "12-15 minutes";
      } else {
        // Default case if categories cannot be determined
        return "10-15 minutes";
      }
    },

    clearNotifications() {
      const userName = localStorage.getItem("userName");
      if (userName) {
        const userNotificationsKey = `user_notifications_${userName}`;
        localStorage.removeItem(userNotificationsKey);
        this.notifications = [];
        
        // Reset unread count
        this.unreadCount = 0;
        eventBus.notificationsCount = 0;
        localStorage.setItem("unread_notifications", 0);
        
        // Dispatch event to notify other components
        window.dispatchEvent(new CustomEvent("notificationUpdated"));
        
        console.log("All notifications cleared");
      }
    },

    initWebSocket() {
      // Use the same host as the API
      const wsUrl = `ws://${window.location.hostname}:8000/ws/orders`;
      
      // Only create a new connection if we don't have one already or if it's closed
      if (!this.ws || this.ws.readyState === WebSocket.CLOSED) {
        console.log('Initializing WebSocket connection in UserNotifications...');
        this.ws = new WebSocket(wsUrl);
        
        this.ws.onopen = () => {
          console.log('WebSocket connected in UserNotifications');
          this.wsConnected = true;
          this.wsReconnectAttempts = 0; // Reset reconnect attempts on successful connection
        };
        
        this.ws.onmessage = (event) => {
          try {
            const data = JSON.parse(event.data);
            console.log('WebSocket message received in UserNotifications:', data);
            
            const userName = localStorage.getItem("userName");
            let notificationAdded = false;
            let isForCurrentUser = false;
            
            // Pre-update badge to ensure UI is responsive
            if (data.target_user === userName) {
              isForCurrentUser = true;
              this.unreadCount += 1;
              eventBus.notificationsCount = this.unreadCount;
              localStorage.setItem("unread_notifications", this.unreadCount);
            }

            // Process different message types
            if (data.type === 'new_order') {
              // Handle new order notification
              if (data.order.customer_name === userName) {
                console.log('New order created, but notification is disabled');
                // No notification will be created for new orders
                // The notification that was here has been removed as requested
              }
            } else if (data.type === 'order_status_update') {
              // Handle order status update notification
              if (data.customer_name === userName) {
                console.log('Processing order status update notification');
                if (data.status === 'completed') {
                  notificationAdded = this.addNewNotification({
                    orderId: data.order_id,
                    message: `Your order #${data.order_id} is now completed! Thank you for your order.`,
                    timestamp: new Date().toISOString(),
                    read: false
                  });
                  
                  // Show notification panel if it's not already open
                  if (!this.isPanelOpen && notificationAdded) {
                    this.showNewNotificationAlert();
                  }
                } else if (data.status === 'declined') {
                  notificationAdded = this.addNewNotification({
                    orderId: data.order_id,
                    message: `Your order #${data.order_id} has been declined. ${data.message || 'We apologize for any inconvenience.'}`,
                    timestamp: new Date().toISOString(),
                    read: false
                  });
                  
                  // Show notification panel if it's not already open
                  if (!this.isPanelOpen && notificationAdded) {
                    this.showNewNotificationAlert();
                  }
                }
              }
            } else if (data.type === 'admin_notification') {
              // Handle direct notifications from admin
              const targetUser = data.target_user;
              
              // Only process the notification if it's for the current user
              if (targetUser === userName) {
                console.log('Processing user notification from admin');
                
                // Special handling for order adjustments (send for approval)
                if (data.action === 'order_adjustment') {
                  console.log('Processing order adjustment notification');
                  
                  // Make sure the adjustment notification has the requiresApproval flag
                  const adjustmentNotification = {
                    ...data.notification,
                    requiresApproval: true,  // Explicitly set to true to show approve/decline buttons
                    read: false
                  };
                  
                  notificationAdded = this.addNewNotification(adjustmentNotification);
                  
                  // Open notification panel automatically to show the approval request
                  if (notificationAdded) {
                    this.showNewNotificationAlert();
                    this.isPanelOpen = true;  // Auto-open the panel for important approvals
                  }
                } else {
                  // Process regular notification
                  const notification = {
                    ...data.notification,
                    read: false // Mark as unread by default
                  };
                  
                  notificationAdded = this.addNewNotification(notification);
                  
                  // Show notification panel if it's not already open
                  if (!this.isPanelOpen && notificationAdded) {
                    this.showNewNotificationAlert();
                  }
                }
              }
            } else if (data.type === 'user_notification') {
              // Handle direct notifications to the user
              const targetUser = data.target_user;
              
              // Only process if it's for the current user
              if (targetUser === userName) {
                console.log('Processing user_notification:', data);
                
                // Handle different action types
                if (data.action === 'order_declined') {
                  console.log('Processing order_declined notification from user_notification type');
                  
                  // Extract notification data
                  const declinedNotification = {
                    ...data.notification,
                    read: false
                  };
                  
                  // Add the notification
                  notificationAdded = this.addNewNotification(declinedNotification);
                  
                  // Show visual feedback and open panel
                  if (notificationAdded) {
                    this.showSuccessMessage = true;
                    this.successMessage = `Order #${declinedNotification.orderId} was declined by the admin`;
                    
                    // Auto-remove the message after 3 seconds
                    setTimeout(() => {
                      this.showSuccessMessage = false;
                    }, 3000);
                    
                    // Open notification panel automatically
                    this.isPanelOpen = true;
                    
                    // Show notification alert
                    this.showNewNotificationAlert();
                    
                    // Force refresh the notifications to ensure UI is updated
                    this.fetchNotifications();
                    
                    // Force update UI
                    this.$forceUpdate();
                  }
                } else if (data.action === 'order_adjustment') {
                  // Already handled above, but kept for completeness
                  console.log('Processing order_adjustment from user_notification type');
                  
                  const adjustmentNotification = {
                    ...data.notification,
                    requiresApproval: true,
                    read: false
                  };
                  
                  notificationAdded = this.addNewNotification(adjustmentNotification);
                  
                  if (notificationAdded) {
                    this.showNewNotificationAlert();
                    this.isPanelOpen = true;
                  }
                } else {
                  // Handle other types of user notifications
                  const notification = {
                    ...data.notification,
                    read: false
                  };
                  
                  notificationAdded = this.addNewNotification(notification);
                  
                  if (notificationAdded && !this.isPanelOpen) {
                    this.showNewNotificationAlert();
                  }
                }
              }
            } else if (data.type === 'order_declined') {
              // Special handling for direct declined orders in real-time
              if (data.customer_name === userName) {
                console.log('Processing order declined notification in real-time:', data);
                
                // Create a clear notification for the user
                const declinedNotification = {
                  orderId: data.order_id,
                  message: `Your order #${data.order_id} has been declined by the admin. ${data.reason || 'We apologize for any inconvenience.'}`,
                  timestamp: new Date().toISOString(),
                  read: false
                };
                
                // Force add this notification
                notificationAdded = this.addNewNotification(declinedNotification);
                console.log('Decline notification added:', notificationAdded);
                
                // Show visual feedback
                this.showSuccessMessage = true;
                this.successMessage = `Order #${data.order_id} was declined by the admin`;
                
                // Auto-remove the message after 3 seconds
                setTimeout(() => {
                  this.showSuccessMessage = false;
                }, 3000);
                
                // Open the notification panel automatically
                this.isPanelOpen = true;
                
                // Show notification alert with sound
                this.showNewNotificationAlert();
                
                // Force refresh the notifications from localStorage to ensure they appear
                this.fetchNotifications();
                
                // Force update the UI
                this.$forceUpdate();
              }
            }
            
            // If we pre-updated the badge but didn't actually add a notification,
            // rollback the count
            if (isForCurrentUser && !notificationAdded) {
              console.log('Notification not added, rolling back badge count');
              this.fetchAndRefreshBadge();
            }
          } catch (error) {
            console.error('Error processing WebSocket message in UserNotifications:', error);
          }
        };
        
        this.ws.onclose = () => {
          console.log('WebSocket disconnected in UserNotifications');
          this.wsConnected = false;
          
          // Implement fixed reconnection attempt with a 5 second delay
          // This matches the approach in NotificationsPage.vue
          setTimeout(() => {
            if (!this.wsConnected) {
              console.log('Attempting to reconnect WebSocket in UserNotifications...');
              this.initWebSocket();
            }
          }, 5000);
        };
        
        this.ws.onerror = (error) => {
          console.error('WebSocket error in UserNotifications:', error);
          this.wsConnected = false;
          
          // Try to reconnect after error with the same approach as NotificationsPage
          setTimeout(() => {
            if (!this.wsConnected) {
              console.log('Attempting to reconnect WebSocket after error in UserNotifications...');
              this.initWebSocket();
            }
          }, 5000);
        };
      }
    },

    formatItems(items) {
      return items.map(item => `${item.name} x${item.quantity}`).join(", ");
    },

    calculateTotal(items) {
      return items.reduce((sum, item) => sum + (item.price * item.quantity), 0).toFixed(2);
    },

    markAllAsRead() {
      const userName = localStorage.getItem("userName");
      if (userName) {
        const userNotificationsKey = `user_notifications_${userName}`;
        
        // Get existing notifications
        let notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
        
        // Mark all notifications as read
        notifications = notifications.map(notification => ({
          ...notification,
          read: true
        }));
        
        // Save back to localStorage
        localStorage.setItem(userNotificationsKey, JSON.stringify(notifications));
        
        // Update the notifications in the component
        this.notifications = notifications;
        
        // Immediately reset unread count to 0
        this.unreadCount = 0;
        
        // Update eventBus to notify other components
        eventBus.notificationsCount = 0;
        
        // Update localStorage for persistence
        localStorage.setItem("unread_notifications", 0);
        
        // Close the confirmation popup
        this.showMarkReadConfirmation = false;
        
        // Force UI to update
        this.$forceUpdate();
        
        // Dispatch event to notify other components
        window.dispatchEvent(new CustomEvent("notificationUpdated"));
        
        console.log('All notifications marked as read');
      }
    },

    // Add new method to show notification alert
    showNewNotificationAlert() {
      // Play notification sound immediately
      try {
        const audio = new Audio('/notification-sound.mp3');
        audio.play().catch(e => console.log('Error playing notification sound:', e));
      } catch(e) {
        console.log('Error playing notification sound:', e);
      }
      
      // Immediately apply animations
      this.$nextTick(() => {
        // Add animation class to notification badge
        const badge = document.querySelector('.floating-notification-badge');
        if (badge) {
          // Remove existing animation if present
          badge.classList.remove('notification-pulse');
          
          // Force reflow to restart animation
          void badge.offsetWidth;
          
          // Add animation class
          badge.classList.add('notification-pulse');
          
          // Remove animation class after it completes
          setTimeout(() => {
            badge.classList.remove('notification-pulse');
          }, 2000);
        }
        
        // Flash the notification button to make it more noticeable
        const notificationButton = document.querySelector('.floating-notification');
        if (notificationButton) {
          notificationButton.classList.add('notification-button-highlight');
          setTimeout(() => {
            notificationButton.classList.remove('notification-button-highlight');
          }, 2000);
        }
        
        console.log('Notification alert displayed immediately');
      });
    },

    // New method to force refresh the badge when needed
    fetchAndRefreshBadge() {
      this.fetchNotifications();
      
      // Force DOM update of badge after fetching notifications
      this.$nextTick(() => {
        const unreadCount = this.notifications.filter(notification => !notification.read).length;
        this.unreadCount = unreadCount;
        eventBus.notificationsCount = unreadCount;
        localStorage.setItem("unread_notifications", unreadCount);
        console.log("Badge refreshed with count:", unreadCount);
      });
    },

    // Approve order adjustment
    approveAdjustment(notification, index) {
      if (!notification.orderId) return;
      
      // Mark notification as read and show immediate UI feedback
      notification.read = true;
      notification.requiresApproval = false;
      notification.message = notification.message + " [Approved]";
      
      // Show pending indicator
      this.showLoadingMessage = true;
      this.loadingMessage = 'Sending approval...';
      
      // Update the order in the database to remove the pending approval flag
      fetch(`http://127.0.0.1:8000/orders/${notification.orderId}/update-items`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          items: notification.items, // Keep the adjusted items
          status: "pending", // Keep as pending
          isPendingApproval: false // Remove the pending approval flag
        })
      })
      .then(response => {
        if (!response.ok) {
          throw new Error(`HTTP error! Status: ${response.status}`);
        }
        return response.json();
      })
      .then(() => {
        console.log("Order approved successfully in database");
        
        // Send approval response notification back to admin
        this.sendAdminNotification(
          notification.orderId,
          `The customer has APPROVED the order adjustments. Order ID: ${notification.orderId}`,
          notification.items
        );
        
        // Send an additional direct notification with customer_approval type for real-time updates
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
          this.ws.send(JSON.stringify({
            type: 'customer_approval',
            orderId: notification.orderId,
            items: notification.items,
            approved: true,
            timestamp: new Date().toISOString()
          }));
        }
        
        // Update the user notifications in localStorage
        const userName = localStorage.getItem("userName");
        const userNotificationsKey = `user_notifications_${userName}`;
        let notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
        
        // Update the notification to show it's been approved
        notifications[index] = {
          ...notification,
          requiresApproval: false,
          message: notification.message
        };
        
        localStorage.setItem(userNotificationsKey, JSON.stringify(notifications));
        this.notifications = notifications;
        
        // Update the unread count
        this.updateUnreadCount();
        
        // Remove loading message
        this.showLoadingMessage = false;
        
        // Show success feedback
        this.showSuccessMessage = true;
        this.successMessage = 'Order adjustments approved!';
        
        // Auto-remove the success message after 3 seconds
        setTimeout(() => {
          this.showSuccessMessage = false;
        }, 3000);
      })
      .catch(error => {
        console.error("Error approving order adjustment:", error);
        
        // Remove loading message
        this.showLoadingMessage = false;
        
        // Show error message
        alert("Error approving order adjustment. Please try again.");
        
        // Revert UI changes on error
        notification.requiresApproval = true;
        notification.message = notification.message.replace(" [Approved]", "");
        this.$forceUpdate(); // Force UI update
      });
    },
    
    // Decline order adjustment
    declineAdjustment(notification, index) {
      if (!notification.orderId) return;
      
      // Mark notification as being processed
      notification.isProcessing = true;
      this.$forceUpdate(); // Force UI update
      
      // Show pending indicator
      this.showLoadingMessage = true;
      this.loadingMessage = 'Declining order...';
      
      // Option 1: Decline the entire order
      fetch(`http://127.0.0.1:8000/orders/${notification.orderId}`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ 
          status: "declined",
          items: notification.originalItems || notification.items
        })
      })
      .then(response => {
        if (!response.ok) {
          throw new Error(`HTTP error! Status: ${response.status}`);
        }
        return response.json();
      })
      .then(() => {
        console.log("Order declined successfully in database");
        
        // Send decline response notification back to admin
        this.sendAdminNotification(
          notification.orderId,
          `The customer has DECLINED the order adjustments. Order ID: ${notification.orderId}`,
          notification.originalItems || notification.items
        );
        
        // Send an additional direct notification with customer_approval type for real-time updates
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
          this.ws.send(JSON.stringify({
            type: 'customer_approval',
            orderId: notification.orderId,
            approved: false,
            timestamp: new Date().toISOString()
          }));
        }
        
        // Remove the notification from the user's list
        const userName = localStorage.getItem("userName");
        const userNotificationsKey = `user_notifications_${userName}`;
        let notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
        
        // Remove this notification
        notifications.splice(index, 1);
        
        localStorage.setItem(userNotificationsKey, JSON.stringify(notifications));
        this.notifications = notifications;
        
        // Update the unread count
        this.updateUnreadCount();
        
        // Hide loading message
        this.showLoadingMessage = false;
        
        // Show success feedback
        this.showSuccessMessage = true;
        this.successMessage = 'Order adjustments declined successfully';
        
        // Auto-remove the success message after 3 seconds
        setTimeout(() => {
          this.showSuccessMessage = false;
        }, 3000);
      })
      .catch(error => {
        console.error("Error declining order adjustment:", error);
        
        // Hide loading message
        this.showLoadingMessage = false;
        
        // Clear processing state
        notification.isProcessing = false;
        this.$forceUpdate();
        
        // Show error message
        alert("Error declining order adjustment. Please try again.");
      });
    },
    
    // Helper to send notifications to admin
    sendAdminNotification(orderId, message, items) {
      // Create a notification for admin
      const adminNotification = {
        orderId,
        customerName: "Admin", // Direct to admin
        message,
        timestamp: new Date().toISOString(),
        items,
        isAdminNotification: true
      };
      
      // Save to admin notifications
      const adminNotificationsKey = "user_notifications_Admin";
      let adminNotifications = JSON.parse(localStorage.getItem(adminNotificationsKey)) || [];
      adminNotifications.push(adminNotification);
      localStorage.setItem(adminNotificationsKey, JSON.stringify(adminNotifications));
      
      // Send WebSocket notification if connected
      if (this.ws && this.ws.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({
          type: 'admin_notification',
          action: 'adjustment_response',
          notification: adminNotification
        }));
      }
    },
    
    // Helper to update unread count
    updateUnreadCount() {
      const unreadCount = this.notifications.filter(n => !n.read).length;
      this.unreadCount = unreadCount;
      eventBus.notificationsCount = unreadCount;
      localStorage.setItem("unread_notifications", unreadCount);
    },
  },
  created() {
    // Initialize notifications and badge
    this.fetchAndRefreshBadge();
    
    // Initialize WebSocket connection
    this.initWebSocket();
    
    // Set up event listeners
    window.addEventListener("notificationUpdated", this.fetchAndRefreshBadge);
  },
  mounted() {
    // Force badge refresh after component is mounted
    this.$nextTick(() => {
      this.fetchAndRefreshBadge();
    });
    
    // Check WebSocket connection periodically and reconnect if needed
    this.wsCheckInterval = setInterval(() => {
      if (!this.wsConnected || !this.ws || this.ws.readyState !== WebSocket.OPEN) {
        console.log('WebSocket not connected in UserNotifications. Attempting to reconnect...');
        this.initWebSocket();
      } else {
        // Periodically refresh badge
        this.fetchAndRefreshBadge();
      }
    }, 30000); // Check every 30 seconds
  },
  beforeUnmount() {
    // Clean up event listeners and intervals
    window.removeEventListener("notificationUpdated", this.fetchAndRefreshBadge);
    clearInterval(this.wsCheckInterval);
    
    // Close WebSocket connection
    if (this.ws) {
      this.ws.close();
    }
  }
};
</script>

<style scoped>
/* Global Styles */
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

/* Notification Badge Animation */
@keyframes notification-pulse {
  0% {
    transform: scale(1);
    box-shadow: 0 0 0 0 rgba(229, 79, 112, 0.7);
  }
  
  70% {
    transform: scale(1.2);
    box-shadow: 0 0 0 10px rgba(229, 79, 112, 0);
  }
  
  100% {
    transform: scale(1);
    box-shadow: 0 0 0 0 rgba(229, 79, 112, 0);
  }
}

/* Notification Button Highlight Animation */
@keyframes notification-button-highlight {
  0% {
    transform: scale(1);
    box-shadow: 0 0 0 0 rgba(229, 79, 112, 0.7);
    background-color: #E54F70;
  }
  
  50% {
    transform: scale(1.15);
    box-shadow: 0 0 20px 5px rgba(229, 79, 112, 0.7);
    background-color: #ff7b96;
  }
  
  100% {
    transform: scale(1);
    box-shadow: 0 0 0 0 rgba(229, 79, 112, 0.7);
    background-color: #E54F70;
  }
}

.notification-pulse {
  animation: notification-pulse 0.8s cubic-bezier(0.66, 0, 0, 1) 2;
}

.notification-button-highlight {
  animation: notification-button-highlight 0.8s cubic-bezier(0.66, 0, 0, 1) 2;
}

.notifications-container {
  /* Empty container for the floating button */
}

/* Floating Notification Button */
.floating-notification {
  position: relative; /* Changed from fixed to relative for header positioning */
  top: auto;
  right: auto;
  width: 40px;
  height: 40px;
  background-color: #E54F70;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 18px;
  cursor: pointer;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  z-index: 500;
  transition: all 0.3s ease;
}

.floating-notification:hover {
  transform: scale(1.1);
  background-color: #d33d5e;
}

.floating-notification-badge {
  position: absolute;
  top: -8px;
  right: -8px;
  background-color: red;
  color: white;
  border-radius: 50%;
  font-size: 14px;
  min-width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

/* Notification Panel Overlay */
.notification-panel-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2000;
}

.notifications-card {
  background-color: white;
  border-radius: 12px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 450px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  max-height: 85vh;
  animation: slideIn 0.3s ease;
}

@keyframes slideIn {
  from {
    transform: translateY(50px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

.close-panel {
  background: none;
  border: none;
  color: #666;
  font-size: 24px;
  cursor: pointer;
  padding: 0;
  margin-left: 10px;
}

.close-panel:hover {
  color: #333;
}

.notifications-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px;
  border-bottom: 1px solid #eee;
}

.notifications-header h2 {
  font-size: 20px;
  font-weight: 600;
  color: #333;
}

.notification-controls {
  display: flex;
  align-items: center;
  gap: 10px;
}

.toggle-button {
  background: none;
  border: none;
  font-size: 20px;
  cursor: pointer;
}

.notifications-list {
  overflow-y: auto;
  flex-grow: 1;
  padding: 10px 0;
  max-height: 60vh;
}

.notification-item {
  display: flex;
  padding: 15px 20px;
  border-bottom: 1px solid #f1f1f1;
}

.notification-item:hover {
  background-color: #f9f9f9;
}

.notification-avatar {
  margin-right: 15px;
}

.notification-avatar img {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  object-fit: cover;
}

.notification-content {
  flex: 1;
}

.notification-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 5px;
}

.notification-name {
  font-weight: 600;
  color: #333;
}

.notification-time {
  color: #888;
  font-size: 12px;
}

.notification-message {
  color: #555;
  font-size: 14px;
  line-height: 1.4;
}

.order-details {
  display: block;
  background-color: rgba(229, 79, 112, 0.05);
  padding: 10px;
  border-radius: 6px;
  margin-top: 10px;
  border-left: 3px solid #E54F70;
  color: #555;
  font-size: 13px;
}

.notifications-footer {
  border-top: 1px solid #eee;
  padding: 15px 20px;
}

.view-all-button {
  width: 100%;
  padding: 10px;
  background-color: transparent;
  border: none;
  border-radius: 6px;
  color: #E54F70;
  cursor: pointer;
  font-size: 14px;
  margin-bottom: 15px;
  text-align: center;
}

.view-all-button:hover {
  background-color: rgba(229, 79, 112, 0.05);
}

.action-buttons {
  display: flex;
  justify-content: space-between;
  gap: 10px;
}

.mark-read-btn, .clear-btn {
  flex: 1;
  padding: 8px 0;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  border: none;
  transition: background-color 0.2s;
}

.mark-read-btn {
  background-color: #E54F70;
  color: white;
}

.mark-read-btn:hover {
  background-color: #d33d5e;
}

.clear-btn {
  background-color: #f1f1f1;
  color: #333;
}

.clear-btn:hover {
  background-color: #e1e1e1;
}

.no-notifications {
  text-align: center;
  padding: 30px 20px;
  color: #888;
  font-size: 15px;
}

.confirmation-popup {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2100;
}

.confirmation-content {
  background-color: white;
  padding: 25px;
  border-radius: 12px;
  width: 300px;
  text-align: center;
}

.confirmation-content h3 {
  margin-bottom: 10px;
  color: #333;
}

.confirmation-buttons {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-top: 20px;
}

.confirm-yes-btn, .confirm-no-btn {
  padding: 8px 20px;
  border-radius: 6px;
  border: none;
  cursor: pointer;
  font-size: 14px;
}

.confirm-yes-btn {
  background-color: #E54F70;
  color: white;
}

.confirm-yes-btn:hover {
  background-color: #d33d5e;
}

.confirm-no-btn {
  background-color: #f1f1f1;
  color: #333;
}

.confirm-no-btn:hover {
  background-color: #e1e1e1;
}

/* Dark Mode Styles */
.dark-mode .floating-notification {
  background-color: #444;
}

.dark-mode .floating-notification:hover {
  background-color: #333;
}

.dark-mode .notifications-card {
  background-color: #333;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.3);
}

.dark-mode .notifications-header {
  border-bottom: 1px solid #444;
}

.dark-mode .notifications-header h2 {
  color: #fff;
}

.dark-mode .close-panel {
  color: #aaa;
}

.dark-mode .close-panel:hover {
  color: #fff;
}

.dark-mode .notification-item {
  border-bottom: 1px solid #444;
}

.dark-mode .notification-item:hover {
  background-color: #3a3a3a;
}

.dark-mode .notification-name {
  color: #fff;
}

.dark-mode .notification-message {
  color: #ddd;
}

.dark-mode .order-details {
  background-color: rgba(229, 79, 112, 0.1);
  color: #ddd;
}

.dark-mode .notifications-footer {
  border-top: 1px solid #444;
}

.dark-mode .view-all-button {
  color: #E54F70;
}

.dark-mode .view-all-button:hover {
  background-color: rgba(229, 79, 112, 0.1);
}

.dark-mode .clear-btn {
  background-color: #444;
  color: #ddd;
}

.dark-mode .clear-btn:hover {
  background-color: #555;
}

.dark-mode .confirmation-content {
  background-color: #333;
}

.dark-mode .confirmation-content h3,
.dark-mode .confirmation-content p {
  color: #fff;
}

.dark-mode .confirm-no-btn {
  background-color: #444;
  color: #ddd;
}

.dark-mode .confirm-no-btn:hover {
  background-color: #555;
}

/* Mobile Responsiveness */
@media (max-width: 768px) {
  .floating-notification {
    width: 35px;
    height: 35px;
    font-size: 16px;
  }
  
  .floating-notification-badge {
    min-width: 18px;
    height: 18px;
    font-size: 10px;
  }
  
  .notifications-card {
    max-width: 90%;
  }
}

@media (max-width: 480px) {
  .floating-notification {
    width: 30px;
    height: 30px;
    font-size: 14px;
  }
  
  .floating-notification-badge {
    min-width: 16px;
    height: 16px;
    font-size: 9px;
    top: -5px;
    right: -5px;
  }
  
  .notifications-card {
    max-width: 95%;
    height: 80vh;
  }
  
  .notification-item {
    padding: 12px 15px;
  }
  
  .notification-avatar img {
    width: 35px;
    height: 35px;
  }
}

/* New styles for adjustment details */
.adjustment-details {
  background-color: #fff3f3;
  border-left: 3px solid #ff6b6b;
  padding: 8px 12px;
  margin: 8px 0;
  font-size: 13px;
  border-radius: 4px;
  color: #545454;
  line-height: 1.4;
}

.dark-mode .adjustment-details {
  background-color: #3d2828;
  border-left: 3px solid #ff6b6b;
  color: #e0e0e0;
}

/* Approval buttons styling */
.approval-actions {
  display: flex;
  gap: 10px;
  margin-top: 12px;
}

.approve-btn, .decline-btn {
  padding: 8px 16px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  font-weight: bold;
  font-size: 14px;
  transition: background-color 0.2s;
}

.approve-btn {
  background-color: #4CAF50;
  color: white;
}

.approve-btn:hover {
  background-color: #3e8e41;
}

.decline-btn {
  background-color: #f44336;
  color: white;
}

.decline-btn:hover {
  background-color: #d32f2f;
}

/* New styles for loading and success messages */
.loading-message {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  background-color: #4a4a4a;
  color: white;
  padding: 10px 20px;
  border-radius: 4px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  z-index: 3000;
  animation: fadeIn 0.3s ease;
}

.success-message-floating {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  background-color: #4CAF50;
  color: white;
  padding: 10px 20px;
  border-radius: 4px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  z-index: 3000;
  animation: fadeInOut 3s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translate(-50%, -20px);
  }
  to {
    opacity: 1;
    transform: translate(-50%, 0);
  }
}

@keyframes fadeInOut {
  0% {
    opacity: 0;
    transform: translate(-50%, -20px);
  }
  10% {
    opacity: 1;
    transform: translate(-50%, 0);
  }
  80% {
    opacity: 1;
    transform: translate(-50%, 0);
  }
  100% {
    opacity: 0;
    transform: translate(-50%, -20px);
  }
}
</style>
