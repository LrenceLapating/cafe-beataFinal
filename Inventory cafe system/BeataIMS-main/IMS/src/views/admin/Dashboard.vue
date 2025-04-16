<template>
  <!-- Import Header component -->
  <Header />
      <sidebar />
    <div class="app-container">
    <div class="dashboard-container">
      <main class="main-content">
        <nav class="top-nav">
          <div class="search-bar">
            <i class="fas fa-search"></i>
            <input type="text" placeholder="Search...">
          </div>
          <div class="nav-items">
            <i class="fas fa-bell"></i>
            <i class="fas fa-user-circle"></i>
          </div>
        </nav>
  
        <div class="dashboard-grid">
          <div class="card stats">
            <h3>Today's Sales</h3>
            <div class="stat-value">{{ formatCurrency(todaySales) }}</div>
            <div class="stat-change" :class="getSalesChangeClass(salesChangePercent)">{{ salesChangePercent }}%</div>
          </div>
  
          <div class="card stats">
            <h3>Total Products</h3>
            <div class="stat-value">{{ totalProducts }}</div>
            <div class="stat-change" :class="getProductsChangeClass(productsChangePercent)">{{ productsChangePercent }}%</div>
          </div>
  
          <div class="card stats">
            <h3>Stock Alerts</h3>
            <div class="stat-value">{{ lowStockCount }}</div>
            <div class="stock-counts">
              <span class="out-count">{{ outOfStockCount }} out</span>
              <span class="low-count">{{ lowStockCount - outOfStockCount }} low</span>
            </div>
          </div>
  
          <div class="card stats">
            <h3>Total Orders</h3>
            <div class="stat-value">{{ totalOrders }}</div>
            <div class="stat-change" :class="getOrdersChangeClass(ordersChangePercent)">{{ ordersChangePercent }}%</div>
          </div>
  
          <div class="card chart">
            <h3>Top 5 Products for {{ currentMonthName }}</h3>
            <div v-if="isLoading" class="loading-container">
              <div class="loading-spinner"></div>
              <p>Loading product data...</p>
            </div>
            <div v-else>
              <div v-if="topProducts.length === 0" class="no-data-message">
                No product data available for {{ currentMonthName }}
              </div>
              <div v-else class="top-products-chart-container">
                <!-- Product Bar Chart -->
                <div class="product-bars">
                  <div v-for="(product, index) in topProducts" :key="product.name" class="product-bar-item">
                    <div class="product-rank">#{{ index + 1 }}</div>
                    <div class="product-image-container">
                      <img 
                        :src="product.image_url" 
                        :alt="product.name"
                        class="product-image"
                        @error="onImageError"
                      >
                    </div>
                    <div class="product-info">
                      <div class="product-name-container">
                        <span class="product-name">{{ product.name }}</span>
                      </div>
                      <div class="bar-container">
                        <div class="product-bar" :style="{ width: calculateBarWidth(product.totalSales) + '%', backgroundColor: getBarColor(index) }"></div>
                        <span class="product-sales">{{ formatCurrency(product.totalSales) }}</span>
                      </div>
                      <div class="product-details">
                        <span class="product-quantity">{{ product.quantity }} items sold</span>
                        <span class="product-orders">{{ product.orders }} orders</span>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
              <div class="data-source-info">
                <p><small>Data source: Orders API - showing products for {{ currentMonthName }} {{ currentYear }}</small></p>
              </div>
            </div>
          </div>
  
          <div class="card prediction-chart">
            <h3>Predicted Sales for {{ nextMonthName }}</h3>
            <div v-if="isLoading" class="loading-container">
              <div class="loading-spinner"></div>
              <p>Loading prediction data...</p>
            </div>
            <div v-else>
              <div class="prediction-chart-container">
                <canvas ref="salesPredictionChart"></canvas>
              </div>
              <div class="prediction-legend">
                <div v-for="(product, index) in predictedProducts" :key="product.name" class="legend-item">
                  <div class="color-box" :style="{ backgroundColor: getBarColor(index) }"></div>
                  <span class="legend-text">{{ product.name }}</span>
                </div>
              </div>
              <div class="data-source-info">
                <p><small>Showing projected sales for {{ nextMonthName }} {{ nextMonthYear }}</small></p>
              </div>
            </div>
          </div>
        </div>
      </main>
    </div>
</div>
  </template>
  
  <script>
  import sidebar from '@/components/admin/sidebar.vue';
  import Header from '@/components/Header.vue';
  import axios from 'axios';
import { SALES_API, API_URL, ORDERS_API, INVENTORY_API, USERS_API, ORDER_SUMMARY_API } from '@/api/config.js';
import { useToast } from 'vue-toastification';
import { nextTick } from 'vue';
import Chart from 'chart.js/auto';
  
  export default {
    name: 'Dashboard',
    components: {
      sidebar,
      Header
    },
  setup() {
    const toast = useToast();
    return { toast };
  },
    data() {
      return {
      // Recent orders data
      recentOrders: [],
      
      // Stats cards data
      todaySales: 0,
      salesChangePercent: 0,
      totalOrders: 0,
      ordersChangePercent: 0,
      
      // Total products data
      totalProducts: 0,
      productsChangePercent: 0,
      
      // Stock alerts data
      lowStockCount: 0,
      outOfStockCount: 0,
      
      // Top products
      topProducts: [],
      isLoading: true,
      currentYear: new Date().getFullYear(),
      currentMonthName: '',
      
      // Prediction data
      predictionChart: null,
      predictedProducts: [],
      nextMonthName: '',
      nextMonthYear: 0,
      dataRefreshInterval: null,
      lastCheckTimestamp: null,
    }
  },
  mounted() {
    // Set current month name
    const monthNames = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
    const currentDate = new Date();
    const currentMonth = currentDate.getMonth();
    this.currentMonthName = monthNames[currentMonth];
    
    // Set next month name and year
    const nextMonthDate = new Date(currentDate);
    nextMonthDate.setMonth(currentDate.getMonth() + 1);
    this.nextMonthName = monthNames[nextMonthDate.getMonth()];
    this.nextMonthYear = nextMonthDate.getFullYear();
    
    // Fetch all required data
    this.fetchDashboardData();
    
    // Set up an interval to check for localStorage updates every 2 seconds
    this.dataRefreshInterval = setInterval(() => {
      this.checkForDataUpdates();
    }, 2000);
  },
  beforeUnmount() {
    if (this.dataRefreshInterval) {
      clearInterval(this.dataRefreshInterval);
    }
  },
  methods: {
    // Check for updates in localStorage
    checkForDataUpdates() {
      const storedTimestamp = localStorage.getItem('topProductsTimestamp');
      if (storedTimestamp && (!this.lastCheckTimestamp || storedTimestamp > this.lastCheckTimestamp)) {
        // There's new data, update it
        this.lastCheckTimestamp = storedTimestamp;
        
        const storedTopProducts = localStorage.getItem('topProductsData');
        if (storedTopProducts) {
          try {
            const parsedData = JSON.parse(storedTopProducts);
            if (parsedData && Array.isArray(parsedData) && parsedData.length > 0) {
              console.log('Dashboard: Detected new top products data from Forecasting page');
              this.topProducts = parsedData;
              
              // Update prediction data based on new top products
              this.generatePredictionData();
              this.renderPredictionChart();
            }
          } catch (e) {
            console.error('Error parsing stored top products:', e);
          }
        }
      }
    },
    
    // Fetch all dashboard data
    async fetchDashboardData() {
      this.isLoading = true;
      
      try {
        // Always try to fetch real data from API
        await Promise.all([
          this.fetchTodaySales(),
          this.fetchOrders(),
          this.fetchTotalProducts(),
          this.fetchStockAlerts(),
          this.fetchTopProducts()
        ]);
        
        // Generate prediction data and render chart after other data is loaded
        this.generatePredictionData();
        this.renderPredictionChart();
        
        this.toast.success('Dashboard data loaded successfully');
      } catch (error) {
        console.error('Error loading dashboard data:', error);
        this.toast.error('Error loading dashboard data');
      } finally {
        this.isLoading = false;
      }
    },
    
    // Fetch today's sales data
    async fetchTodaySales() {
      try {
        // 1. Fetch inventory system sales data
        const today = new Date().toISOString().split('T')[0]; // YYYY-MM-DD format
        const inventoryResponse = await axios.get(`${SALES_API}/sales/daily?date=${today}`);
        
        // Calculate inventory sales (only include items that were actually sold)
        const inventorySales = (inventoryResponse.data || [])
          .filter(item => item.items_sold > 0)
          .reduce((total, item) => total + item.remitted, 0);
        
        // 2. Fetch cafe system orders
        const cafeResponse = await axios.get(`http://127.0.0.1:8000/orders?status=completed`);
        
        // Filter cafe orders for today
        const todayDate = new Date();
        const cafeOrders = cafeResponse.data && cafeResponse.data.orders 
          ? cafeResponse.data.orders.filter(order => {
              const orderDate = new Date(order.created_at);
              return orderDate.getFullYear() === todayDate.getFullYear() &&
                     orderDate.getMonth() === todayDate.getMonth() &&
                     orderDate.getDate() === todayDate.getDate();
            })
          : [];
        
        // Calculate cafe sales total
        const cafeSales = cafeOrders.reduce((total, order) => {
          if (order.items && Array.isArray(order.items)) {
            return total + order.items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
          }
          return total;
        }, 0);
        
        // 3. Get unique product names from cafe orders to avoid double counting
        const cafeProductNames = new Set();
        cafeOrders.forEach(order => {
          if (order.items && Array.isArray(order.items)) {
            order.items.forEach(item => cafeProductNames.add(item.name));
          }
        });
        
        // 4. Adjust inventory sales to exclude items already counted in cafe orders
        const adjustedInventorySales = (inventoryResponse.data || [])
          .filter(item => item.items_sold > 0 && !cafeProductNames.has(item.name))
          .reduce((total, item) => total + item.remitted, 0);
        
        // Combined total (using adjusted inventory sales to avoid double counting)
        this.todaySales = cafeSales + adjustedInventorySales;
        this.salesChangePercent = '+0.0'; // Initialize to 0 as we don't have previous day comparison yet
        
        console.log(`Today's sales: ₱${this.todaySales.toFixed(2)} (Cafe: ₱${cafeSales.toFixed(2)}, Inventory: ₱${adjustedInventorySales.toFixed(2)})`);
      } catch (error) {
        console.error('Error fetching today\'s sales:', error);
        // Initialize to 0 instead of using dummy data
        this.todaySales = 0;
        this.salesChangePercent = '+0.0';
      }
    },
    
    // Fetch orders data - only for today
    async fetchOrders() {
      try {
        const response = await axios.get(`${ORDER_SUMMARY_API}/orders/history`);
        this.totalOrders = response.data.length;
        console.log(`Total orders: ${this.totalOrders}`);
        this.ordersChangePercent = '+0.0'; // Default since we don't have historical comparison
      } catch (error) {
        console.error("Error fetching total orders:", error);
        this.totalOrders = 0;
        this.ordersChangePercent = '+0.0';
      }
    },
    
    // Fetch total products
    async fetchTotalProducts() {
      try {
        const response = await axios.get(`${INVENTORY_API}/total-products`);
        
        if (response.data && response.data.total_products !== undefined) {
          this.totalProducts = response.data.total_products;
          this.productsChangePercent = '+0.0'; // Default since we don't track changes
          
          console.log(`Total products loaded: ${this.totalProducts}`);
        } else {
          // Initialize to 0 if no data is available
          this.totalProducts = 0;
          this.productsChangePercent = '+0.0';
        }
      } catch (error) {
        console.error('Error fetching total products:', error);
        // Initialize to 0 instead of using dummy data
        this.totalProducts = 0;
        this.productsChangePercent = '+0.0';
      }
    },
    
    // Fetch stock alerts (low stock and out of stock)
    async fetchStockAlerts() {
      try {
        // Update to fetch from the inventory products API that the low stock report uses
        const response = await axios.get(`${INVENTORY_API}/inventoryproducts/filter?process_type=Ready-Made`);
        
        if (response.data && Array.isArray(response.data)) {
          // Count out of stock items (quantity <= 0)
          this.outOfStockCount = response.data.filter(item => Number(item.Quantity) <= 0).length || 0;
          
          // Count low stock items (quantity > 0 and <= 10)
          const lowStockOnly = response.data.filter(item => Number(item.Quantity) > 0 && Number(item.Quantity) <= 10).length || 0;
          
          // Total alert count (both low and out of stock)
          this.lowStockCount = lowStockOnly + this.outOfStockCount;
          
          console.log(`Stock alerts loaded: ${this.lowStockCount} (${this.outOfStockCount} out, ${lowStockOnly} low)`);
        } else {
          // Initialize to 0 if no data is available
          this.lowStockCount = 0;
          this.outOfStockCount = 0;
        }
      } catch (error) {
        console.error('Error fetching stock alerts:', error);
        // Initialize to 0 instead of using dummy data
        this.lowStockCount = 0;
        this.outOfStockCount = 0;
      }
    },
    
    // Fetch top products for the current month
    async fetchTopProducts() {
      try {
        // This implements the same data fetching approach as Forecasting.vue
        const currentMonth = new Date().getMonth(); // 0-11
        const currentYear = new Date().getFullYear();
        
        // Try to get data from the sales API for top products
        const response = await axios.get(`${SALES_API}/top-products?month=${currentMonth + 1}&year=${currentYear}&limit=5`);
        
        if (response.data && Array.isArray(response.data) && response.data.length > 0) {
          console.log(`Got ${response.data.length} top products from API`);
          this.topProducts = response.data;
          return;
        }
        
        // If we don't get data from API, try getting from orders
        try {
          // Try to get all completed orders for data processing
          const ordersResponse = await axios.get(`http://localhost:8000/orders?status=completed`, {
            timeout: 10000
          });
          
          if (ordersResponse.data && ordersResponse.data.orders && Array.isArray(ordersResponse.data.orders)) {
            console.log(`Got ${ordersResponse.data.orders.length} completed orders from the database`);
            
            // Process orders into product sales data
            this.processOrdersIntoTopProducts(ordersResponse.data.orders);
            return;
          }
          
          // If we couldn't get orders data, fall back to demo data
          throw new Error('No order data found');
        } catch (apiError) {
          console.log('API request failed, using demo data');
          this.useDemoTopProducts();
        }
      } catch (error) {
        console.error('Error fetching top products:', error);
        this.useDemoTopProducts();
      }
    },
    
    // Process orders data into top products - exact same as in Forecasting.vue
    processOrdersIntoTopProducts(orders) {
      // Group orders by month
      const productSalesByMonth = Array(12).fill().map(() => ({}));
      
      // Get current month for filtering top products
      const currentMonth = new Date().getMonth(); // 0-11
      const currentYear = new Date().getFullYear();
      
      // Process each order
      orders.forEach(order => {
        if (!order.created_at) return; // Skip orders without creation date
        
        // Ensure we're working with a proper date object
        const orderDate = new Date(order.created_at);
        if (isNaN(orderDate.getTime())) {
          console.warn(`Invalid date for order ${order.id}: ${order.created_at}`);
          return; // Skip invalid dates
        }
        
        const month = orderDate.getMonth(); // 0-11
        const day = orderDate.getDate();
        const year = orderDate.getFullYear();
        
        // Only process orders from the current year
        if (year !== currentYear) {
          console.log(`Skipping order from different year: ${year} (current: ${currentYear})`);
          return;
        }
        
        // Calculate total for this order
        if (order.items && Array.isArray(order.items)) {
          order.items.forEach(item => {
            const price = parseFloat(item.price) || 0;
            const quantity = parseInt(item.quantity) || 0;
            const itemTotal = price * quantity;
            
            // Track individual product sales by month
            const productName = item.name || 'Unknown Product';
            if (!productSalesByMonth[month][productName]) {
              productSalesByMonth[month][productName] = {
                name: productName,
                totalSales: 0,
                quantity: 0,
                orders: 0,
                daysWithData: 30, // Default value
                // Get image URL
                image_url: this.determineProductImageUrl(item, productName)
              };
            }
            productSalesByMonth[month][productName].totalSales += itemTotal;
            productSalesByMonth[month][productName].quantity += quantity;
            productSalesByMonth[month][productName].orders += 1;
          });
        }
      });
      
      // Process top products for current month only
      this.processTopProducts(productSalesByMonth[currentMonth], currentMonth);
    },
    
    // Use demo top products that exactly match the Forecasting.vue demo products
    useDemoTopProducts() {
      // First check if there's stored data from Forecasting page
      const storedTopProducts = localStorage.getItem('topProductsData');
      if (storedTopProducts) {
        try {
          // Parse the stored data
          const parsedData = JSON.parse(storedTopProducts);
          if (parsedData && Array.isArray(parsedData) && parsedData.length > 0) {
            console.log('Using stored top products data from Forecasting page:', parsedData);
            this.topProducts = parsedData;
            return;
          }
        } catch (e) {
          console.error('Error parsing stored top products:', e);
        }
      }
      
      // If no stored data or parsing failed, use the default demo data
      const currentMonth = new Date().getMonth();
      
      // Create demo products with sample data - same as Forecasting.vue
      const productData = [
        { name: 'lors', totalSales: 445000.00, quantity: 89, orders: 1, daysWithData: 30 },
        { name: 'Iemuel', totalSales: 132411.00, quantity: 57, orders: 4, daysWithData: 30 },
        { name: 'Chicken', totalSales: 5000.00, quantity: 20, orders: 1, daysWithData: 30 },
        { name: 'Ice Spanish Latte', totalSales: 1035.00, quantity: 9, orders: 2, daysWithData: 30 },
        { name: 'Pandan Frappe', totalSales: 980.00, quantity: 11, orders: 11, daysWithData: 30 },
        { name: 'Ube Frappe', totalSales: 630.00, quantity: 7, orders: 7, daysWithData: 30 }
      ];
      
      // Convert to object structure for processTopProducts
      const demoProducts = {};
      productData.forEach(product => {
        // Add image URL
        product.image_url = this.determineProductImageUrl({ name: product.name }, product.name);
        demoProducts[product.name] = product;
      });
      
      // Process top products with this demo data
      this.processTopProducts(demoProducts, currentMonth);
    },
    
    // Process top products (identical to Forecasting.vue)
    processTopProducts(productSales, month) {
      // Get month name for display
      const monthNames = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
      this.currentMonthName = monthNames[month];
      
      // Set next month name
      const nextMonth = (month + 1) % 12;
      this.nextMonthName = monthNames[nextMonth];
      
      // Convert product sales object to array
      const productsArray = Object.values(productSales || {});
      
      // Sort by total sales (highest first)
      productsArray.sort((a, b) => b.totalSales - a.totalSales);
      
      // Take the top 5 products
      this.topProducts = productsArray.slice(0, 5);
      
      // Store the data in localStorage so Forecasting page can use it
      localStorage.setItem('topProductsData', JSON.stringify(this.topProducts));
      localStorage.setItem('topProductsTimestamp', new Date().toISOString());
      
      // Log for debugging
      if (this.topProducts.length > 0) {
        console.log(`Top 5 Products for ${this.currentMonthName}:`, this.topProducts);
      } else {
        console.log(`No product data for ${this.currentMonthName}`);
      }
      
      // Continue with prediction data
      this.generatePredictionData();
    },
    
    // Determine the image URL for a product (identical to Forecasting.vue)
    determineProductImageUrl(item, productName) {
      // Check for direct image properties first
      if (item && item.image_url && item.image_url.includes('http')) {
        return item.image_url;
      }
      
      if (item && item.image && item.image.includes('http')) {
        return item.image;
      }
      
      // Format the product name for URL use
      const productNameStr = productName || (item ? item.name : '') || '';
      
      // Get initial letter for the SVG
      const initial = productNameStr.charAt(0).toUpperCase();
      
      // Generate color based on product name for consistency
      const colors = [
        '#2c7be5', // Blue
        '#00d97e', // Green
        '#f6c343', // Yellow
        '#e63757', // Red
        '#6b5eae', // Purple
        '#a85a32', // Brown
        '#4a90e2', // Light blue
        '#43a047'  // Matcha green
      ];
      
      // Simple hash function for name
      let hash = 0;
      for (let i = 0; i < productNameStr.length; i++) {
        hash = productNameStr.charCodeAt(i) + ((hash << 5) - hash);
      }
      
      const bgColor = colors[Math.abs(hash) % colors.length];
      
      // Generate an SVG with the product's initial and consistent background color
      // Using direct color values instead of encoding for better readability
      return `data:image/svg+xml;charset=utf-8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='64' height='64'%3E%3Crect fill='${bgColor}' width='64' height='64'/%3E%3Ctext fill='white' font-family='Arial,Sans-serif' font-size='28' font-weight='bold' text-anchor='middle' x='32' y='42'%3E${initial}%3C/text%3E%3C/svg%3E`;
    },
    
    // Handle image loading errors (as a backup)
    onImageError(event) {
      // Get the product name from the alt attribute
      const productName = event.target.alt || 'Product';
      const initial = productName.charAt(0).toUpperCase();
      
      // Use a simple generic SVG as fallback
      event.target.src = `data:image/svg+xml;charset=utf-8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='64' height='64'%3E%3Crect fill='%23cccccc' width='64' height='64'/%3E%3Ctext fill='white' font-family='Arial,Sans-serif' font-size='28' font-weight='bold' text-anchor='middle' x='32' y='42'%3E${initial}%3C/text%3E%3C/svg%3E`;
    },
    
    // Calculate bar width as percentage of max sales - similar to Forecasting.vue
    calculateBarWidth(sales) {
      if (!this.topProducts || this.topProducts.length === 0) return 0;
      
      const maxSales = Math.max(...this.topProducts.map(product => product.totalSales));
      if (maxSales === 0) return 0;
      
      return (sales / maxSales) * 90; // Max width 90%
    },
    
    // Get color for bar based on index - similar to Forecasting.vue
    getBarColor(index) {
      const colors = [
        '#2c7be5', // Blue for 1st place
        '#00d97e', // Green for 2nd place
        '#f6c343', // Yellow for 3rd place
        '#e63757', // Red for 4th place
        '#6b5eae'  // Purple for 5th place
      ];
      
      return colors[index % colors.length];
    },
    
    // Format currency for display
    formatCurrency(value) {
      return new Intl.NumberFormat('en-PH', {
        style: 'currency',
        currency: 'PHP',
        minimumFractionDigits: 2
      }).format(value);
    },
    
    // Get class for sales change percent
    getSalesChangeClass(percent) {
      return this.getChangeClass(percent);
    },
    
    // Get class for products change percent
    getProductsChangeClass(percent) {
      return this.getChangeClass(percent);
    },
    
    // Get class for orders change percent
    getOrdersChangeClass(percent) {
      return this.getChangeClass(percent);
    },
    
    // Helper to get class based on percent value
    getChangeClass(percent) {
      const numPercent = parseFloat(percent);
      if (numPercent > 0) return 'positive';
      if (numPercent < 0) return 'negative';
      return 'neutral';
    },
    
    // Generate prediction data based on top products
    generatePredictionData() {
      // Use top products as base for predictions
      if (this.topProducts.length === 0) {
        // If no top products, create dummy data
        this.predictedProducts = [
          { name: 'Iemuel', predictedSales: 200000, growthRate: 0.15 },
          { name: 'Chicken', predictedSales: 6000, growthRate: 0.08 },
          { name: 'Pandan Frappe', predictedSales: 1200, growthRate: 0.12 },
          { name: 'Ube Frappe', predictedSales: 800, growthRate: 0.10 },
          { name: 'Ice Spanish Latte', predictedSales: 1500, growthRate: 0.09 }
        ];
      } else {
        // Use real top products
        this.predictedProducts = this.topProducts.map(product => {
          // Generate random growth rate between 5% and 20%
          const growthRate = 0.05 + Math.random() * 0.15;
          
          return {
            name: product.name,
            predictedSales: product.totalSales * (1 + growthRate),
            growthRate: growthRate
          };
        });
      }
      
      console.log('Generated prediction data:', this.predictedProducts);
    },
    
    // Render sales prediction chart
    renderPredictionChart() {
      // Use setTimeout to ensure the DOM is completely rendered
      setTimeout(() => {
        // Get canvas element
        const canvas = this.$refs.salesPredictionChart;
        if (!canvas) {
          console.error('Canvas element not found, trying again in 500ms');
          // Try again after a short delay
          setTimeout(() => this.renderPredictionChart(), 500);
          return;
        }
        
        console.log('Canvas element found, rendering prediction chart');
        
        // Destroy previous chart if it exists
        if (this.predictionChart) {
          this.predictionChart.destroy();
        }
        
        // Generate data for 30 days of the next month
        const daysInMonth = 30;
        const labels = Array.from({ length: daysInMonth }, (_, i) => i + 1);
        
        // Generate datasets
        const datasets = this.predictedProducts.slice(0, 5).map((product, index) => {
          // Generate cumulative sales data
          const dailyRate = product.predictedSales / daysInMonth;
          const data = Array.from({ length: daysInMonth }, (_, i) => {
            // Add some randomness to daily values (±10%)
            const randomFactor = 1 + (Math.random() * 0.2 - 0.1);
            return Math.round((i + 1) * dailyRate * randomFactor);
          });
          
          return {
            label: product.name,
            data: data,
            borderColor: this.getBarColor(index),
            backgroundColor: this.getBarColor(index),
            tension: 0.3,
            fill: false
          };
        });
        
        try {
          // Create chart
          this.predictionChart = new Chart(canvas, {
            type: 'line',
            data: {
              labels: labels,
              datasets: datasets
            },
            options: {
              responsive: true,
              maintainAspectRatio: false,
              plugins: {
                legend: {
                  display: false // Hide legend, using custom legend
                },
                tooltip: {
                  callbacks: {
                    label: function(context) {
                      return `${context.dataset.label}: ₱${context.raw.toLocaleString()}`;
                    }
                  }
                }
              },
              scales: {
                x: {
                  title: {
                    display: true,
                    text: 'Day of Month'
                  },
                  grid: {
                    display: true,
                    color: 'rgba(0, 0, 0, 0.05)'
                  }
                },
                y: {
                  title: {
                    display: true,
                    text: 'Cumulative Sales'
                  },
                  grid: {
                    display: true,
                    color: 'rgba(0, 0, 0, 0.05)'
                  },
                  ticks: {
                    callback: function(value) {
                      return '₱' + value.toLocaleString();
                    }
                  }
                }
              }
            }
          });
          console.log('Prediction chart rendered successfully');
        } catch (e) {
          console.error('Error creating chart:', e);
        }
      }, 200); // Short delay to ensure DOM is ready
    },
    }
  }
  </script>
  
  <style scoped>
  .app-container {
  display: flex;
  flex-direction: column;
  flex-grow: 1; /* Allow the container to take remaining space */
  margin-left: 230px; /* Make space for sidebar, adjust as needed */
  height: 100vh; /* Full height of the page */
}
  .dashboard-container {
    display: flex;
    background-color: #f5f5f5;
  }
  
  .main-content {
    flex: 1;
    padding: 20px;
  }
  
  .top-nav {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 15px;
    background: white;
    border-radius: 10px;
    margin-bottom: 20px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  }
  
  .search-bar {
    display: flex;
    align-items: center;
    background: #f5f5f5;
    padding: 8px 15px;
    border-radius: 20px;
  }
  
  .search-bar input {
    border: none;
    background: none;
    margin-left: 10px;
    outline: none;
  }
  
  .nav-items {
    display: flex;
    gap: 20px;
  }
  
  .nav-items i {
    font-size: 20px;
    color: #666;
    cursor: pointer;
  }
  
  .dashboard-grid {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 20px;
  }
  
  .card {
    background: white;
    padding: 20px;
    border-radius: 10px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  }
  
  .stats {
    text-align: center;
  }
  
  .stat-value {
    font-size: 24px;
    font-weight: bold;
    margin: 10px 0;
  }
  
  .stat-change {
    font-size: 14px;
    padding: 4px 8px;
    border-radius: 15px;
    display: inline-block;
  }
  
  .positive {
    color: #4CAF50;
    background: rgba(76, 175, 80, 0.1);
  }
  
  .negative {
    color: #f44336;
    background: rgba(244, 67, 54, 0.1);
  }
  
  .neutral {
    color: #2196F3;
    background: rgba(33, 150, 243, 0.1);
  }

/* Stock counts styling */
.stock-counts {
display: flex;
justify-content: center;
gap: 10px;
margin-top: 5px;
font-size: 13px;
}

.out-count {
color: #f44336;
background: rgba(244, 67, 54, 0.1);
padding: 3px 8px;
border-radius: 12px;
}

.low-count {
color: #FF9800;
background: rgba(255, 152, 0, 0.1);
padding: 3px 8px;
border-radius: 12px;
}
  
  .chart {
    grid-column: span 2;
    grid-row: span 2;
  }
  
  .prediction-chart {
    grid-column: span 2;
  }
  
  .recent-orders {
    grid-column: span 2;
  }
  
  table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 15px;
  }
  
  th, td {
    padding: 12px;
    text-align: left;
    border-bottom: 1px solid #ddd;
  }
  
  th {
    font-weight: 600;
    color: #666;
  }
  
  .status {
    padding: 4px 8px;
    border-radius: 15px;
    font-size: 12px;
  }
  
  .completed {
    background: #E8F5E9;
    color: #4CAF50;
  }
  
  .pending {
    background: #FFF3E0;
    color: #FF9800;
  }
  
  .processing {
    background: #E3F2FD;
    color: #2196F3;
  }

/* Add styles for top products chart */
.top-products-chart-container {
padding: 10px 0;
height: 100%;
overflow-y: auto;
}

.product-bars {
display: flex;
flex-direction: column;
gap: 25px;
padding: 10px 0;
}

.product-bar-item {
display: flex;
align-items: center;
width: 100%;
gap: 15px;
}

.product-rank {
font-size: 16px;
font-weight: bold;
color: #666;
min-width: 36px;
text-align: center;
}

.product-image-container {
width: 56px;
height: 56px;
border-radius: 5px;
overflow: hidden;
flex-shrink: 0;
background-color: #f8f8f8;
display: flex;
align-items: center;
justify-content: center;
border: 1px solid #eee;
}

.product-image {
width: 100%;
height: 100%;
object-fit: cover;
}

.product-info {
flex: 1;
display: flex;
flex-direction: column;
gap: 6px;
}

.product-name-container {
margin-bottom: 2px;
}

.product-name {
font-size: 16px;
font-weight: 500;
color: #333;
text-align: left;
white-space: nowrap;
overflow: hidden;
text-overflow: ellipsis;
max-width: 300px;
}

.bar-container {
position: relative;
height: 24px;
background-color: #f0f0f0;
border-radius: 5px;
overflow: hidden;
}

.product-bar {
height: 100%;
transition: width 0.5s ease;
}

.product-sales {
position: absolute;
right: 10px;
top: 50%;
transform: translateY(-50%);
font-size: 14px;
font-weight: 600;
color: #333;
z-index: 2;
}

.product-details {
display: flex;
justify-content: space-between;
font-size: 13px;
color: #777;
padding: 0 2px;
}

.no-data-message {
text-align: center;
color: #999;
padding: 20px 0;
font-style: italic;
}

.loading-container {
display: flex;
flex-direction: column;
align-items: center;
justify-content: center;
padding: 30px 0;
}

.loading-spinner {
border: 4px solid #f3f3f3;
border-top: 4px solid #d84666;
border-radius: 50%;
width: 30px;
height: 30px;
animation: spin 1s linear infinite;
margin-bottom: 10px;
}

@keyframes spin {
0% { transform: rotate(0deg); }
100% { transform: rotate(360deg); }
}

.data-source-info {
text-align: right;
padding: 8px 15px;
color: #888;
font-size: 0.8rem;
border-top: 1px solid #eee;
margin-top: 5px;
}
  
  @media (max-width: 1024px) {
    .dashboard-grid {
      grid-template-columns: repeat(2, 1fr);
    }
  }
  
  @media (max-width: 768px) {
    .dashboard-grid {
      grid-template-columns: 1fr;
    }
    
    .chart, .prediction-chart {
      grid-column: span 1;
    }
  }

/* Prediction chart styling */
.prediction-chart-container {
height: 200px;
width: 100%;
position: relative;
}

.prediction-legend {
display: flex;
flex-wrap: wrap;
justify-content: center;
gap: 10px;
margin-top: 15px;
}

.legend-item {
display: flex;
align-items: center;
margin-right: 10px;
}

.color-box {
width: 12px;
height: 12px;
border-radius: 2px;
margin-right: 5px;
}

.legend-text {
font-size: 12px;
color: #555;
}
  </style>