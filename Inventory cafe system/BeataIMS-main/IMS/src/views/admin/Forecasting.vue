<template>
  <Header />
  <sidebar />
  <div class="app-container">
    <div class="sales-container">
      <div class="header-section">
        <h2 class="section-title">{{ currentMonthName }} Sales</h2>
      </div>

      <div class="dashboard-row">
        <div class="card stats-card">
          <h3>{{ currentMonthName }} Sales</h3>
          <div class="stat-value">{{ formatCurrency(currentMonthSales) }}</div>
        </div>

        <div class="card stats-card">
          <h3>{{ currentMonthName }} Orders</h3>
          <div class="stat-value">{{ currentMonthOrders }}</div>
        </div>

        <div class="card stats-card">
          <h3>Average Order Value</h3>
          <div class="stat-value">{{ formatCurrency(averageOrderValue) }}</div>
        </div>
      </div>

      <div class="charts-section">
        <div class="chart-card card">
          <h3>
            Monthly Sales
            <span class="scale-toggle" @click="toggleChartScale">
              {{ useLogScale ? 'Switch to Normal Scale' : 'Switch to Log Scale' }}
            </span>
          </h3>
          <div class="chart-container">
            <canvas ref="monthlySalesChart"></canvas>
          </div>
          </div>
        </div>
        
      <!-- Top Products Bar Chart -->
      <div class="top-products-section">
        <div class="top-products-card card">
          <h3>Top 10 Products for {{ currentMonthName }}</h3>
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
                      :src="product.fallback_image || getFixedImageUrl(product)"
                      :alt="product.name"
                      class="product-image"
                      @error="onImageError"
                    >
          </div>
                  <div class="product-info">
                    <div class="product-name-container">
                      <span class="product-name">
                        {{ cleanProductName(product.name || product.id || 'Product') }}
                        <span v-if="product.name && product.name.startsWith('Unnamed')" class="unnamed-note">(ID: {{ product.id || 'Unknown' }})</span>
                      </span>
                      <span v-if="product.source === 'combined'" class="combined-badge">Combined</span>
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
              <p><small>Data source: Cafe Beata & Inventory Systems - showing combined products for {{ currentMonthName }} {{ currentYear }}</small></p>
            </div>
          </div>
          </div>
        </div>
        
      <!-- Predictions Chart -->
      <div class="predictions-section">
        <div class="predictions-card card">
          <h3>Sales Predictions for {{ nextMonthName }}</h3>
          <div v-if="isLoading" class="loading-container">
            <div class="loading-spinner"></div>
            <p>Generating predictions...</p>
                    </div>
          <div v-else>
            <div v-if="predictedProducts.length === 0" class="no-data-message">
              No prediction data available for {{ nextMonthName }}
                  </div>
            <div v-else>
              <!-- Product Predictions Summary -->
              <div class="prediction-summary">
                <p>Based on current trends, these products are predicted to lead sales in {{ nextMonthName }}:</p>
                <div class="prediction-products">
                  <div v-for="(product, index) in predictedProducts" :key="product.name" class="prediction-product">
                    <span class="prediction-rank">#{{ index + 1 }}</span>
                    <span class="prediction-name">{{ cleanProductName(product.name) }}</span>
                    <span class="prediction-value">{{ formatCurrency(product.predictedSales) }}</span>
                  </div>
        </div>
      </div>

              <!-- Predictions Chart -->
              <div class="chart-container">
                <canvas ref="predictionsChart"></canvas>
        </div>
              
              <div class="prediction-info">
                <p>
                  <small>Forecast methodology: Analysis of current sales patterns, product seasonality, and market trends from both Cafe Beata and Inventory systems.</small>
                </p>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Chart from 'chart.js/auto';
import sidebar from '@/components/admin/sidebar.vue';
import Header from '@/components/Header.vue';
import axios from 'axios';
import { SALES_API, API_URL, REPORTS_API } from '@/api/config.js';
import { useToast } from 'vue-toastification';
import { nextTick } from 'vue';

// Global Chart.js configuration
Chart.defaults.font.family = "'Arial', sans-serif";
Chart.defaults.color = '#555';
Chart.defaults.responsive = true;
Chart.defaults.maintainAspectRatio = false;

export default {
  name: 'Forecasting',
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
      isSidebarCollapsed: false,
      monthlyData: [],
      totalYearlySales: 0,
      averageMonthlySales: 0,
      totalOrders: 0,
      monthlySalesChart: null,
      topProducts: [],
      isLoading: true,
      currentYear: new Date().getFullYear(),
      currentMonthName: '',
      currentMonthSales: 0,
      currentMonthOrders: 0,
      averageOrderValue: 0,
      predictedProducts: [],
      nextMonthName: '',
      predictionsChart: null,
      useLogScale: true, // Start with log scale by default
    };
  },
  mounted() {
    this.loading = true;
    
    // Check if there's a test mode flag in the URL for demo purposes
    const urlParams = new URLSearchParams(window.location.search);
    if (urlParams.get('demo') === 'true') {
      // Use demo data directly
      this.createDemoData();
      return;
    }
  
    // Only fetch real data, with no demo data fallback
    this.fetchRealSalesData()
      .then(() => {
        // Initialize UI with the data
        this.calculateStatistics();
        this.preloadProductImages();
        
        // Safely initialize charts after data is loaded
        this.safeInitializeCharts();
      })
      .catch(error => {
        console.error("Error initializing monthly sales data:", error);
        this.toast.error('Could not fetch sales data. Please check the database connection.');
      })
      .finally(() => {
        this.loading = false;
      });
  },
  methods: {
    // Fetch only real sales data from the database with no fallbacks to demo data
    async fetchRealSalesData() {
      this.isLoading = true;
      try {
        console.log("Fetching real data from both Cafe Beata and Inventory systems...");
        
        // Get data from both systems
        const cafePromise = axios.get(`http://localhost:8000/orders?status=completed`, {
          timeout: 10000
        });
        
        const inventorySalesPromise = axios.get(`http://localhost:8001/api/sales/forecasting/historical-sales`, {
          timeout: 10000
        });
        
        // Also get top products from inventory system
        const currentMonth = new Date().getMonth() + 1; // API expects 1-12 for months
        const currentYear = new Date().getFullYear();
        const inventoryProductsPromise = axios.get(
          `http://localhost:8001/api/sales/top-products?month=${currentMonth}&year=${currentYear}&limit=10`, 
          { timeout: 10000 }
        );
        
        // Wait for all requests to complete
        const [cafeResponse, inventorySalesResponse, inventoryProductsResponse] = 
          await Promise.allSettled([cafePromise, inventorySalesPromise, inventoryProductsPromise]);
        
        // Process Cafe Beata data (if available)
        let cafeOrders = [];
        if (cafeResponse.status === 'fulfilled' && cafeResponse.value.data && 
            cafeResponse.value.data.orders && Array.isArray(cafeResponse.value.data.orders)) {
          console.log(`Got ${cafeResponse.value.data.orders.length} completed orders from Cafe Beata`);
          cafeOrders = cafeResponse.value.data.orders;
        } else {
          console.log("Could not fetch data from Cafe Beata system or no orders found");
        }
        
        // Process Inventory system sales data (if available)
        let inventorySales = [];
        if (inventorySalesResponse.status === 'fulfilled' && inventorySalesResponse.value.data && 
            Array.isArray(inventorySalesResponse.value.data)) {
          console.log(`Got ${inventorySalesResponse.value.data.length} days of sales data from Inventory system`);
          inventorySales = inventorySalesResponse.value.data;
        } else {
          console.log("Could not fetch sales data from Inventory system or no sales found");
        }
        
        // Process Inventory top products (if available)
        let inventoryProducts = [];
        if (inventoryProductsResponse.status === 'fulfilled' && inventoryProductsResponse.value.data && 
            Array.isArray(inventoryProductsResponse.value.data)) {
          console.log(`Got ${inventoryProductsResponse.value.data.length} top products from Inventory system`);
          inventoryProducts = inventoryProductsResponse.value.data;
        } else {
          console.log("Could not fetch top products from Inventory system");
        }
        
        // If we have data from any source, process it
        if (cafeOrders.length > 0 || inventorySales.length > 0 || inventoryProducts.length > 0) {
          // Process sales data from Inventory system first for the chart
          if (inventorySales.length > 0) {
            this.processInventorySalesData(inventorySales);
          }
          
          // Use real inventory product data if available
          const currentMonth = new Date().getMonth(); // 0-11
          if (inventoryProducts.length > 0) {
            this.processInventoryTopProducts(inventoryProducts, currentMonth);
          }
          
          // Then process orders from Cafe Beata system
          if (cafeOrders.length > 0) {
            this.processOrdersIntoMonthlySales(cafeOrders, 'cafe');
          }
          
          // Calculate statistics after processing all data
          this.calculateStatistics();
          this.preloadProductImages();
          this.toast.success('Successfully loaded sales data from both systems');
          return;
        }
                
        // If we couldn't get data from either system, initialize with empty data
        console.log("No data from either system, showing empty data");
        this.initializeEmptyData();
        this.toast.error('Could not find sales data. Showing empty data.');
        return;
      } catch (error) {
        console.error("Failed to fetch real sales data:", error);
        this.toast.error('Database connection error. Showing empty data.');
        
        // Initialize with empty data
        this.initializeEmptyData();
        return;
      } finally {
        this.isLoading = false;
      }
    },
    
    // Process orders data into monthly sales - using only real data
    processOrdersIntoMonthlySales(orders, source = 'cafe') {
      console.log(`Processing real orders data from ${source} into monthly sales...`);
      
      // Group orders by month
      const monthlyTotals = new Array(12).fill(0);
      const monthlyCounts = new Array(12).fill(0);
      const daysWithData = new Array(12).fill(0);
      const uniqueDays = new Array(12).fill().map(() => new Set());
      const monthlyOrders = new Array(12).fill(0);  // Track actual order counts per month
      
      // Track product sales for top products table - by month
      const productSalesByMonth = Array(12).fill().map(() => ({}));
      
      // Get current month for filtering top products
      const currentMonth = new Date().getMonth(); // 0-11
      
      // Process each order
      orders.forEach(order => {
        if (!order.created_at) return; // Skip orders without creation date
        
        // Ensure we're working with a proper date object
        const orderDate = new Date(order.created_at);
        if (isNaN(orderDate.getTime())) {
          console.warn(`Invalid date for order ${order.id}: ${order.created_at}`);
          return; // Skip invalid dates
        }
        
        // Log the parsed date for debugging
        console.log(`Processing order from ${orderDate.toISOString()}, ID: ${order.id}, created_at: ${order.created_at}`);
        
        const month = orderDate.getMonth(); // 0-11
        const day = orderDate.getDate();
        const year = orderDate.getFullYear();
        const currentYear = new Date().getFullYear();
        
        // Only process orders from the current year (matching the daily sales report logic)
        if (year !== currentYear) {
          console.log(`Skipping order from different year: ${year} (current: ${currentYear})`);
          return;
        }
        
        // Add this day to unique days for this month
        uniqueDays[month].add(day);
        
        // Calculate total for this order
        let orderTotal = 0;
        if (order.items && Array.isArray(order.items)) {
          orderTotal = order.items.reduce((total, item) => {
            const price = parseFloat(item.price) || 0;
            const quantity = parseInt(item.quantity) || 0;
            const itemTotal = price * quantity;
            
            // Track individual product sales by month
            const productName = item.name || `Unnamed Product (ID: ${item.id || 'Unknown'})`;
            if (!productSalesByMonth[month][productName]) {
              productSalesByMonth[month][productName] = {
                name: productName,
                totalSales: 0,
                quantity: 0,
                orders: 0,
                source: source,
                // Get image URL - try multiple possible sources and formats
                image_url: this.determineProductImageUrl(item, productName)
              };
            }
            productSalesByMonth[month][productName].totalSales += itemTotal;
            productSalesByMonth[month][productName].quantity += quantity;
            productSalesByMonth[month][productName].orders += 1;
            
            return total + itemTotal;
          }, 0);
          
          // Log each order's calculated total
          console.log(`Order ${order.id} from ${orderDate.toLocaleDateString()} total: ₱${orderTotal.toFixed(2)}`);
        }
        
        // Add to monthly total
        monthlyTotals[month] += orderTotal;
        monthlyCounts[month]++;
        monthlyOrders[month]++;  // Increment order count for this month
      });
      
      // Count unique days with data per month
      for (let i = 0; i < 12; i++) {
        daysWithData[i] = uniqueDays[i].size;
      }
      
      // Log the monthly summaries for debugging
      console.log("Monthly summaries:");
      const monthNames = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
      for (let i = 0; i < 12; i++) {
        if (monthlyTotals[i] > 0 || monthlyOrders[i] > 0) {
          console.log(`${monthNames[i]}: ₱${monthlyTotals[i].toFixed(2)}, ${monthlyOrders[i]} orders, ${daysWithData[i]} days with data`);
        }
      }
      
      // Create or update the monthly data array with real values
      if (!this.monthlyData || this.monthlyData.length === 0) {
      this.monthlyData = monthNames.map((month, index) => ({
        month: month,
        sales: monthlyTotals[index],
        daysWithData: daysWithData[index],
        orderCount: monthlyOrders[index]  // Store actual order count
      }));
      } else {
        // Add to existing data
        for (let i = 0; i < 12; i++) {
          this.monthlyData[i].sales += monthlyTotals[i];
          this.monthlyData[i].orderCount += monthlyOrders[i];
          this.monthlyData[i].daysWithData = Math.max(this.monthlyData[i].daysWithData, daysWithData[i]);
        }
      }
      
      // Process top products for current month only
      this.processTopProducts(productSalesByMonth[currentMonth], currentMonth, source);
    },
    
    // Process real top products data from the inventory system
    processInventoryTopProducts(products, month) {
      console.log("Processing real top products from inventory system:", products);
      
      // Get month name for display
      const monthNames = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
      this.currentMonthName = monthNames[month];
      
      // Set next month name
      const nextMonth = (month + 1) % 12;
      this.nextMonthName = monthNames[nextMonth];
      
      // Convert to format expected by the rest of the app
      let unnamedCount = 0;
      const formattedProducts = products.map(product => {
        // Log original product for debugging
        console.log("Raw product data:", product);
        
        // Try to find the product name from different possible fields
        let name = null;
        
        // Check various common name fields
        if (product.name && product.name.trim() !== '') {
          name = product.name;
        } else if (product.title && product.title.trim() !== '') {
          name = product.title;
        } else if (product.product_name && product.product_name.trim() !== '') {
          name = product.product_name;
        } else if (product.item_name && product.item_name.trim() !== '') {
          name = product.item_name;
        }
        
        // Try to extract name from image URL if no name is found but image URL exists
        if (!name && product.image_url) {
          try {
            // Extract filename from URL
            const urlParts = product.image_url.split('/');
            let fileName = urlParts[urlParts.length - 1];
            
            // Remove extension
            fileName = fileName.split('.')[0];
            
            // Replace underscores and hyphens with spaces
            fileName = fileName.replace(/[_-]/g, ' ');
            
            // Capitalize each word
            fileName = fileName.split(' ').map(word => 
              word.charAt(0).toUpperCase() + word.slice(1).toLowerCase()
            ).join(' ');
            
            if (fileName) {
              name = fileName;
              console.log(`Extracted name from image URL: ${name}`);
            }
          } catch (e) {
            console.error('Error extracting name from image URL:', e);
          }
        }
        
        // If still no name found, use unnamed product with incrementing number
        if (!name) {
          name = `Unnamed Product ${++unnamedCount}`;
          console.warn(`No name found for product, using fallback: ${name}`, product);
        }
        
        // Ensure the image URL is properly handled
        let image_url = product.image_url;
        
        // If we have an image URL but it needs processing
        if (image_url && !image_url.startsWith('http') && !image_url.startsWith('data:')) {
          if (image_url.startsWith('/')) {
            image_url = `http://localhost:8001${image_url}`;
          } else {
            image_url = `http://localhost:8001/uploads/products/${image_url}`;
          }
        }
        
        return {
          name: name,
          totalSales: product.totalSales,
          quantity: product.quantity,
          orders: product.orders,
          daysWithData: product.daysWithData || 1,
          source: 'inventory',
          image_url: image_url || this.determineProductImageUrl(null, name)
        };
      });
      
      // Sort by total sales (highest first)
      formattedProducts.sort((a, b) => b.totalSales - a.totalSales);
      
      // Store in topProducts or merge with existing
      if (!this.topProducts || this.topProducts.length === 0) {
        this.topProducts = formattedProducts.slice(0, 10);
      } else {
        // Merge with existing top products, combining any with the same name
        let mergedProducts = [...this.topProducts];
        
        // For each new product
        formattedProducts.forEach(newProduct => {
          // Normalize names for comparison (lowercase and trim)
          const normalizedNewName = newProduct.name.toLowerCase().trim();
          
          // Check if this product already exists in our list
          const existingProductIndex = mergedProducts.findIndex(p => 
            p.name.toLowerCase().trim() === normalizedNewName
          );
          
          if (existingProductIndex >= 0) {
            // Product already exists, merge the data
            const existingProduct = mergedProducts[existingProductIndex];
            
            // Combine the sales data
            existingProduct.totalSales += newProduct.totalSales;
            existingProduct.quantity += newProduct.quantity;
            existingProduct.orders += newProduct.orders;
            existingProduct.daysWithData = Math.max(existingProduct.daysWithData, newProduct.daysWithData || 0);
            
            // Mark as combined product
            existingProduct.source = 'combined';
            
            // Update the product in the array
            mergedProducts[existingProductIndex] = existingProduct;
          } else {
            // This is a new product, add it to the array
            mergedProducts.push(newProduct);
          }
        });
        
        // Sort by total sales again
        mergedProducts.sort((a, b) => b.totalSales - a.totalSales);
        
        // Keep only the top 5
        this.topProducts = mergedProducts.slice(0, 5);
      }
      
      // Store the data in localStorage
      localStorage.setItem('topProductsData', JSON.stringify(this.topProducts));
      localStorage.setItem('topProductsTimestamp', new Date().toISOString());
      
      // Log for debugging
      console.log(`Top products after adding inventory products:`, this.topProducts);
      
      // Generate prediction data
      this.generatePredictionData();
    },
    
    // Process sales data from Inventory system into monthly format - for chart data only
    processInventorySalesData(salesData) {
      console.log("Processing Inventory system sales data for charts...");
      
      // Monthly totals and counts
      const monthlyTotals = new Array(12).fill(0);
      const daysWithData = new Array(12).fill(0);
      const uniqueDays = new Array(12).fill().map(() => new Set());
      
      // Current month and year
      const currentMonth = new Date().getMonth();
      const currentYear = new Date().getFullYear();
      
      // Process each day's sales data
      salesData.forEach(day => {
        if (!day.date) return;
        
        // Parse the date
        const saleDate = new Date(day.date);
        if (isNaN(saleDate.getTime())) {
          console.warn(`Invalid date in inventory sales: ${day.date}`);
          return;
        }
        
        const month = saleDate.getMonth();
        const dayOfMonth = saleDate.getDate();
        const year = saleDate.getFullYear();
        
        // Only process current year
        if (year !== currentYear) {
          return;
        }
        
        // Add to unique days
        uniqueDays[month].add(dayOfMonth);
        
        // Add sales to monthly total
        const salesAmount = parseFloat(day.sales) || 0;
        monthlyTotals[month] += salesAmount;
      });
      
      // Count unique days with data
      for (let i = 0; i < 12; i++) {
        daysWithData[i] = uniqueDays[i].size;
      }
      
      // Log monthly summaries
      console.log("Inventory system monthly summaries:");
      const monthNames = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
      for (let i = 0; i < 12; i++) {
        if (monthlyTotals[i] > 0) {
          console.log(`${monthNames[i]}: ₱${monthlyTotals[i].toFixed(2)}, ${daysWithData[i]} days with data`);
        }
      }
      
      // Create or update monthly data
      if (!this.monthlyData || this.monthlyData.length === 0) {
        this.monthlyData = monthNames.map((month, index) => ({
          month: month,
          sales: monthlyTotals[index],
          daysWithData: daysWithData[index],
          orderCount: daysWithData[index] // Use days with data as proxy for order count
        }));
      } else {
        // Add to existing data
        for (let i = 0; i < 12; i++) {
          this.monthlyData[i].sales += monthlyTotals[i];
          this.monthlyData[i].daysWithData = Math.max(this.monthlyData[i].daysWithData, daysWithData[i]);
          // We don't have actual order counts from inventory system, so we don't update orderCount
        }
      }
    },
    
    // Determine the image URL for a product
    determineProductImageUrl(item, productName) {
      // Check for direct image properties first
      if (item && item.image_url && item.image_url.includes('http')) {
        return item.image_url;
      }
      
      if (item && item.image && item.image.includes('http')) {
        return item.image;
      }
      
      // Get product name for the fallback image
      const productNameStr = productName || (item ? item.name : '') || '';
      
      // Return a fallback image using our helper
      return this.createFallbackImage(productNameStr);
    },
    
    // Initialize with empty data for current year
    initializeEmptyData() {
      console.log("Initializing empty data structure...");
      
      const monthNames = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
      const fullMonthNames = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
      const currentMonth = new Date().getMonth();
      
      // Set current month name
      this.currentMonthName = fullMonthNames[currentMonth];
      
      this.monthlyData = [];
      
      // Add all months with zero sales
      for (let i = 0; i <= 11; i++) {
        this.monthlyData.push({
          month: monthNames[i],
          sales: 0,
          daysWithData: 0,
          orderCount: 0  // Initialize order count as zero
        });
      }
      
      // Check if we have stored top products data
      const storedTopProducts = localStorage.getItem('topProductsData');
      if (storedTopProducts) {
        try {
          // Parse the stored data
          const parsedData = JSON.parse(storedTopProducts);
          if (parsedData && Array.isArray(parsedData) && parsedData.length > 0) {
            console.log('Using stored top products data:', parsedData);
            this.topProducts = parsedData;
          } else {
            // Empty top products array if no valid stored data
            this.topProducts = [];
          }
        } catch (e) {
          console.error('Error parsing stored top products:', e);
          this.topProducts = [];
        }
      } else {
        // Empty top products array if no stored data
        this.topProducts = [];
      }
      
      // Reset current month data
      this.currentMonthSales = 0;
      this.currentMonthOrders = 0;
      this.averageOrderValue = 0;
      
      // Calculate statistics
      this.calculateStatistics();
    },
    
    // Calculate statistics based on monthly data
    calculateStatistics() {
      // Total yearly sales (from real data only)
      this.totalYearlySales = this.monthlyData.reduce((sum, month) => sum + month.sales, 0);
      
      // Average monthly sales (only counting months with data)
      const monthsWithData = this.monthlyData.filter(month => month.daysWithData > 0);
      this.averageMonthlySales = monthsWithData.length > 0 ? 
        this.totalYearlySales / monthsWithData.length : 0;
      
      // Calculate total orders based on actual order counts, not an estimated average value
      this.totalOrders = this.monthlyData.reduce((sum, month) => sum + month.orderCount, 0);
      
      // Calculate current month data
      const currentMonthIndex = new Date().getMonth();
      this.currentMonthSales = this.monthlyData[currentMonthIndex] ? this.monthlyData[currentMonthIndex].sales : 0;
      this.currentMonthOrders = this.monthlyData[currentMonthIndex] ? this.monthlyData[currentMonthIndex].orderCount : 0;
      
      // Calculate average order value with safety check for division by zero
      this.averageOrderValue = this.currentMonthOrders > 0 ? 
        this.currentMonthSales / this.currentMonthOrders : 0;
    },
    
    // Render the monthly sales chart
    renderMonthlySalesChart() {
      try {
      const canvas = this.$refs.monthlySalesChart;
        if (!canvas) {
        console.warn('Monthly sales chart canvas not found');
          return;
        }
        
        // Check if the canvas is in the DOM
        if (!document.body.contains(canvas)) {
          console.warn('Monthly sales chart canvas is not in the DOM');
          return;
        }
        
      // Destroy existing chart if it exists
      if (this.monthlySalesChart) {
        this.monthlySalesChart.destroy();
          this.monthlySalesChart = null;
      }
      
        // Wait a bit to ensure DOM is ready
        setTimeout(() => {
          try {
      // Prepare data for the chart
      const months = this.monthlyData.map(item => item.month);
      const salesData = this.monthlyData.map(item => item.sales);
      
      // Get current month to differentiate between actual and incomplete months
      const currentMonth = new Date().getMonth(); // 0-11
      
      // Better color palette - more vibrant and distinct colors
      const barColors = this.monthlyData.map((item, index) => {
        if (index < currentMonth) {
          // Past months - use a gradient of blues
          return `rgba(53, 162, 235, ${0.7 + (index / (currentMonth + 1)) * 0.3})`;
        } else if (index === currentMonth) {
          // Current month - highlight in green
          return 'rgba(75, 192, 92, 0.9)';
        } else {
          // Future months - gray them out
          return 'rgba(201, 203, 207, 0.5)';
        }
      });
      
      // Create the chart
      this.monthlySalesChart = new Chart(canvas, {
        type: 'bar',
          data: {
          labels: months,
          datasets: [{
            label: 'Monthly Sales',
            data: salesData,
            backgroundColor: barColors,
            borderColor: barColors.map(color => color.replace('0.8', '1').replace('0.9', '1').replace('0.7', '1')),
            borderWidth: 1
          }]
          },
          options: {
            responsive: true,
            maintainAspectRatio: false,
            scales: {
              y: {
                    type: this.useLogScale ? 'logarithmic' : 'linear',
                beginAtZero: true,
                    min: this.useLogScale ? 1000 : 0, // Minimum value depends on scale type
                ticks: {
                callback: (value) => this.formatCurrency(value)
              }
            }
          },
            plugins: {
              legend: {
              display: false // Hide the legend since we only have one dataset
              },
              tooltip: {
                callbacks: {
                label: (context) => {
                  const index = context.dataIndex;
                  const monthData = this.monthlyData[index];
                  const salesValue = monthData.sales;
                  const daysCount = monthData.daysWithData;
                  
                  if (daysCount === 0) {
                    return 'No sales data available for this month';
                  }
                  
                  // For formatted values
                  const formattedSales = this.formatCurrency(salesValue);
                  
                  return [
                    `Sales: ${formattedSales}`,
                    `Orders: ${monthData.orderCount}`,
                    `Days with data: ${daysCount}`
                  ];
                }
                }
              }
            }
          }
        });
          } catch (e) {
            console.error('Error rendering monthly sales chart:', e);
          }
        }, 100);
      } catch (e) {
        console.error('Error in renderMonthlySalesChart:', e);
      }
    },
    
    // Format currency for display
    formatCurrency(value) {
      return new Intl.NumberFormat('en-PH', {
        style: 'currency',
        currency: 'PHP',
        minimumFractionDigits: 2
      }).format(value);
    },
    
    // Process top products (modified to handle multiple sources)
    processTopProducts(productSales, month, source = 'cafe') {
      console.log(`Processing top products from ${source} for month ${month}`);
      
      // Get month name for display
      const monthNames = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
      this.currentMonthName = monthNames[month];
      
      // Set next month name
      const nextMonth = (month + 1) % 12;
      this.nextMonthName = monthNames[nextMonth];
      
      // Convert product sales object to array and validate names
      let unnamedCount = 0;
      const productsArray = Object.values(productSales || {}).map(product => {
        // Add detailed logging for debugging
        console.log(`Processing product: ${JSON.stringify(product)}`);
        
        // Try to find the product name from different possible fields
        if (!product.name || product.name.trim() === '') {
          // Check various common name fields
          if (product.title && product.title.trim() !== '') {
            product.name = product.title;
          } else if (product.product_name && product.product_name.trim() !== '') {
            product.name = product.product_name;
          } else if (product.item_name && product.item_name.trim() !== '') {
            product.name = product.item_name;
          } else if (product.image_url) {
            // Try to extract name from image URL
            try {
              // Extract filename from URL
              const urlParts = product.image_url.split('/');
              let fileName = urlParts[urlParts.length - 1];
              
              // Remove extension
              fileName = fileName.split('.')[0];
              
              // Replace underscores and hyphens with spaces
              fileName = fileName.replace(/[_-]/g, ' ');
              
              // Capitalize each word
              fileName = fileName.split(' ').map(word => 
                word.charAt(0).toUpperCase() + word.slice(1).toLowerCase()
              ).join(' ');
              
              if (fileName) {
                product.name = fileName;
                console.log(`Extracted name from image URL: ${product.name}`);
              } else {
                product.name = `Unnamed Product ${++unnamedCount}`;
              }
            } catch (e) {
              console.error('Error extracting name from image URL:', e);
              product.name = `Unnamed Product ${++unnamedCount}`;
            }
          } else {
            product.name = `Unnamed Product ${++unnamedCount}`;
          }
          
          console.log(`Using name: ${product.name} for product with missing name`);
        }
        return product;
      });
      
      // If this is the first source being processed, set topProducts directly
      // Otherwise, merge with existing topProducts
      if (!this.topProducts || this.topProducts.length === 0) {
      // Sort by total sales (highest first)
      productsArray.sort((a, b) => b.totalSales - a.totalSales);
      
        // Take the top products
        this.topProducts = productsArray.slice(0, 10); // Get more than 5 to allow for merging
      } else {
        // Merge with existing top products, combining any with the same name
        let mergedProducts = [...this.topProducts];
        
        // For each new product
        productsArray.forEach(newProduct => {
          // Normalize names for comparison (lowercase and trim)
          const normalizedNewName = newProduct.name.toLowerCase().trim();
          
          // Check if this product already exists in our list
          const existingProductIndex = mergedProducts.findIndex(p => 
            p.name.toLowerCase().trim() === normalizedNewName
          );
          
          if (existingProductIndex >= 0) {
            // Product already exists, merge the data
            const existingProduct = mergedProducts[existingProductIndex];
            
            // Combine the sales data
            existingProduct.totalSales += newProduct.totalSales;
            existingProduct.quantity += newProduct.quantity;
            existingProduct.orders += newProduct.orders;
            existingProduct.daysWithData = Math.max(existingProduct.daysWithData, newProduct.daysWithData || 0);
            
            // Mark as combined product
            existingProduct.source = 'combined';
            
            // Update the product in the array
            mergedProducts[existingProductIndex] = existingProduct;
          } else {
            // This is a new product, add it to the array
            mergedProducts.push(newProduct);
          }
        });
        
        // Sort by total sales again
        mergedProducts.sort((a, b) => b.totalSales - a.totalSales);
        
        // Keep only the top 5
        this.topProducts = mergedProducts.slice(0, 5);
      }
      
      // Store the data in localStorage
      localStorage.setItem('topProductsData', JSON.stringify(this.topProducts));
      localStorage.setItem('topProductsTimestamp', new Date().toISOString());
      
      // Log for debugging
      if (this.topProducts.length > 0) {
        console.log(`Top products for ${this.currentMonthName} (after ${source} data):`, this.topProducts);
      } else {
        console.log(`No product data for ${this.currentMonthName}`);
      }
      
      // Generate prediction data
      this.generatePredictionData();
    },
    
    // Generate predictions for top products in the next month
    generatePredictionData() {
      if (!this.topProducts || this.topProducts.length === 0) {
        this.predictedProducts = [];
        return;
      }
      
      // Take the current top 5 products as a starting point
      this.predictedProducts = this.topProducts.map(product => {
        // Create a deep copy of the product
        const predictedProduct = { ...product };
        
        // Create sales prediction data with 30 days of the next month
        const totalDays = 30; // Assuming 30 days in a month for simplicity
        
        // Generate daily sales data for the next month
        predictedProduct.dailyPredictions = [];
        
        // Calculate growth factors and trends
        const growthFactor = this.calculateGrowthFactor(product.name);
        const seasonalFactor = this.calculateSeasonalFactor(this.currentMonthName, this.nextMonthName, product.name);
        
        // Base daily sales (current month's average)
        let baseDaily = product.totalSales / (product.daysWithData || 1);
        
        // Apply the growth and seasonal factors to predict next month's total
        const predictedTotal = product.totalSales * growthFactor * seasonalFactor;
        const predictedDaily = predictedTotal / totalDays;
        
        // Create daily data points with some variability to make the chart interesting
        let cumulativeSales = 0;
        
        for (let day = 1; day <= totalDays; day++) {
          // Add some daily variation (higher on weekends, random fluctuations)
          let isWeekend = (day % 7 === 0 || day % 7 === 6);
          let dailyVariation = isWeekend ? 1.2 : 0.9 + Math.random() * 0.2;
          
          // Special factor for certain days (like promotions or events)
          let specialDayFactor = 1.0;
          if (day === 1 || day === 15 || day === totalDays) {
            specialDayFactor = 1.3; // Higher sales on these special days
          }
          
          // Calculate this day's predicted sales
          let daySales = predictedDaily * dailyVariation * specialDayFactor;
          cumulativeSales += daySales;
          
          // Add data point
          predictedProduct.dailyPredictions.push({
            day,
            sales: cumulativeSales
          });
        }
        
        // Update the total predicted sales
        predictedProduct.predictedSales = cumulativeSales;
        
        return predictedProduct;
      });
      
      // Sort by predicted sales (highest first)
      this.predictedProducts.sort((a, b) => b.predictedSales - a.predictedSales);
      
      // Log that prediction data is ready
      console.log('Prediction data generated successfully');
    },
    
    // Calculate growth factor based on product performance
    calculateGrowthFactor(productName) {
      // This would ideally use historical data to calculate trends
      // For now, we use a simple method based on the product type:
      
      const productLower = productName.toLowerCase();
      
      // Coffee products tend to have steady growth
      if (productLower.includes('coffee')) {
        return 1.15 + (Math.random() * 0.1); // 15-25% growth
      }
      
      // Matcha products are trending
      if (productLower.includes('matcha')) {
        return 1.35 + (Math.random() * 0.15); // 35-50% growth
      }
      
      // Frappe products vary more by season
      if (productLower.includes('frappe')) {
        const currentMonth = new Date().getMonth();
        // Higher growth in warmer months (April-September)
        if (currentMonth >= 3 && currentMonth <= 8) {
          return 1.4 + (Math.random() * 0.2); // 40-60% growth
        } else {
          return 1.1 + (Math.random() * 0.1); // 10-20% growth
        }
      }
      
      // Latte products tend to be steady
      if (productLower.includes('latte')) {
        return 1.2 + (Math.random() * 0.1); // 20-30% growth
      }
      
      // Default growth factor (10-30%)
      return 1.1 + (Math.random() * 0.2);
    },
    
    // Calculate seasonal factor based on month
    calculateSeasonalFactor(currentMonth, nextMonth, productName) {
      const productLower = productName.toLowerCase();
      
      // Seasonal factors based on month transitions
      const monthPairs = {
        'January-February': 1.1, // Slight increase after January
        'February-March': 1.15, // Spring rise
        'March-April': 1.2, // Continuing spring rise
        'April-May': 1.15, // Pre-summer
        'May-June': 1.3, // Summer starts
        'June-July': 1.25, // Peak summer
        'July-August': 0.95, // Late summer decrease
        'August-September': 0.9, // End of summer
        'September-October': 0.95, // Fall starts
        'October-November': 1.1, // Holiday season starting
        'November-December': 1.4, // Holiday peak
        'December-January': 0.7, // Post-holiday drop
      };
      
      // Product-specific seasonal factors
      let productFactor = 1.0;
      
      // Hot drinks increase in cold months
      if (productLower.includes('hot') || productLower.includes('coffee') && !productLower.includes('iced')) {
        if (nextMonth === 'November' || nextMonth === 'December' || nextMonth === 'January' || nextMonth === 'February') {
          productFactor = 1.25;
        } else if (nextMonth === 'June' || nextMonth === 'July' || nextMonth === 'August') {
          productFactor = 0.85;
        }
      }
      
      // Cold drinks increase in warm months
      if (productLower.includes('iced') || productLower.includes('frappe') || productLower.includes('cold')) {
        if (nextMonth === 'May' || nextMonth === 'June' || nextMonth === 'July' || nextMonth === 'August') {
          productFactor = 1.45;
        } else if (nextMonth === 'December' || nextMonth === 'January' || nextMonth === 'February') {
          productFactor = 0.75;
        }
      }
      
      // Get base seasonal factor for month transition
      const monthPair = `${currentMonth}-${nextMonth}`;
      const baseFactor = monthPairs[monthPair] || 1.0;
      
      // Combine base and product-specific factors
      return baseFactor * productFactor;
    },
    
    // Render the predictions chart
    renderPredictionsChart() {
      try {
      const canvas = this.$refs.predictionsChart;
      if (!canvas) {
        console.warn('Predictions chart canvas not found');
        return;
      }
        
        // Check if the canvas is in the DOM
        if (!document.body.contains(canvas)) {
          console.warn('Predictions chart canvas is not in the DOM');
        return;
      }
      
      // Destroy existing chart if it exists
      if (this.predictionsChart) {
        this.predictionsChart.destroy();
          this.predictionsChart = null;
      }
      
      // Skip if no predictions
      if (!this.predictedProducts || this.predictedProducts.length === 0) {
          console.warn('No prediction data available');
        return;
      }
      
        // Wait a bit to ensure DOM is ready
        setTimeout(() => {
          try {
      // Prepare line chart datasets
      const datasets = this.predictedProducts.map((product, index) => {
        return {
                label: this.cleanProductName(product.name),
          data: product.dailyPredictions.map(day => ({ x: day.day, y: day.sales })),
          borderColor: this.getBarColor(index),
          backgroundColor: this.getBarColor(index) + '33', // Add transparency
          borderWidth: 2,
          tension: 0.4, // Smoother curves
          fill: false,
          pointRadius: 0, // Hide points for cleaner look
          pointHoverRadius: 5 // Show points on hover
        };
      });
      
      // Create the line chart
      this.predictionsChart = new Chart(canvas, {
        type: 'line',
        data: {
          datasets
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          scales: {
            x: {
              type: 'linear',
              position: 'bottom',
              title: {
                display: true,
                text: 'Day of Month'
              },
              ticks: {
                stepSize: 5
              }
            },
            y: {
              beginAtZero: true,
              title: {
                display: true,
                text: 'Cumulative Sales'
              },
              ticks: {
                callback: (value) => this.formatCurrency(value)
              }
            }
          },
          plugins: {
            legend: {
              position: 'top'
            },
            tooltip: {
              callbacks: {
                label: (context) => {
                  const datasetIndex = context.datasetIndex;
                  const product = this.predictedProducts[datasetIndex];
                  const day = context.parsed.x;
                  const sales = context.parsed.y;
                  
                  return [
                          `${this.cleanProductName(product.name)}`,
                    `Day ${day}: ${this.formatCurrency(sales)}`
                  ];
                }
              }
            },
            title: {
              display: true,
              text: `Predicted Sales for ${this.nextMonthName}`,
              font: {
                size: 16
              }
            }
          }
        }
      });
          } catch (e) {
            console.error('Error rendering predictions chart:', e);
          }
        }, 100);
      } catch (e) {
        console.error('Error in renderPredictionsChart:', e);
      }
    },
    
    // Method to handle image loading errors (as a backup)
    onImageError(event) {
      // Log the error for debugging
      console.error(`Image failed to load: ${event.target.src}`, event);
      
      // Get the product name from the alt attribute
      const productName = event.target.alt || 'Product';
      
      // Use our helper method to create a fallback image
      event.target.src = this.createFallbackImage(productName);
    },
    
    // Get fixed image URL for product
    getFixedImageUrl(product) {
      if (!product) return '';
      
      // For combined products, prefer the image from the inventory system if available
      if (product.source === 'combined' && product.image_url) {
        // If the URL doesn't start with http or data:, it might be a relative path
        if (!product.image_url.startsWith('http') && !product.image_url.startsWith('data:')) {
          if (product.image_url.startsWith('/')) {
            return `http://localhost:8001${product.image_url}`;
          } else {
            return `http://localhost:8001/${product.image_url}`;
          }
        }
        return product.image_url;
      }
      
      // For inventory products, check if the URL is relative
      if (product.source === 'inventory' && product.image_url) {
        if (!product.image_url.startsWith('http') && !product.image_url.startsWith('data:')) {
          // Special handling for inventory product images
          if (product.image_url.startsWith('/uploads/')) {
            return `http://localhost:8001${product.image_url}`;
          } else if (product.image_url.startsWith('uploads/')) {
            return `http://localhost:8001/${product.image_url}`;
          } else if (product.image_url.startsWith('/')) {
            return `http://localhost:8001${product.image_url}`;
          } else {
            return `http://localhost:8001/uploads/products/${product.image_url}`;
          }
        }
        return product.image_url;
      }
      
      // For cafe products or if no image is available
      if (product.image_url) {
        if (!product.image_url.startsWith('http') && !product.image_url.startsWith('data:')) {
          if (product.image_url.startsWith('/')) {
            return `http://localhost:8000${product.image_url}`;
          } else {
            return `http://localhost:8000/${product.image_url}`;
          }
        }
        return product.image_url;
      }
      
      // Create a fallback image using our helper
      return this.createFallbackImage(product.name);
    },
    
    // Get a consistent color based on a string
    getHashColor(str) {
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
      
      let hash = 0;
      for (let i = 0; i < str.length; i++) {
        hash = str.charCodeAt(i) + ((hash << 5) - hash);
      }
      
      return colors[Math.abs(hash) % colors.length];
    },
    
    // Calculate bar width as percentage of max sales
    calculateBarWidth(sales) {
      if (!this.topProducts || this.topProducts.length === 0) return 0;
      
      const maxSales = Math.max(...this.topProducts.map(product => product.totalSales));
      if (maxSales === 0) return 0;
      
      return (sales / maxSales) * 90; // Max width 90%
    },
    
    // Get color for bar based on index
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
    
    // Create demo data for testing
    createDemoData() {
      // Set current month
      const monthNames = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
      const currentMonth = new Date().getMonth();
      this.currentMonthName = monthNames[currentMonth];
      
      // Create demo products with sample data
      const productData = [
        { name: 'lors', totalSales: 445000.00, quantity: 89, orders: 1, daysWithData: 30 },
        { name: 'Iemuel', totalSales: 132411.00, quantity: 57, orders: 4, daysWithData: 30 },
        { name: 'Chicken', totalSales: 5000.00, quantity: 20, orders: 1, daysWithData: 30 },
        { name: 'Ice Spanish Latte', totalSales: 1035.00, quantity: 9, orders: 2, daysWithData: 30 },
        { name: 'Pandan Frappe', totalSales: 980.00, quantity: 11, orders: 11, daysWithData: 30 },
        { name: 'Ube Frappe', totalSales: 630.00, quantity: 7, orders: 7, daysWithData: 30 }
      ];
      
      // Validate product data to ensure all have names
      productData.forEach((product, index) => {
        if (!product.name || product.name.trim() === '') {
          product.name = `Demo Product ${index + 1}`;
        }
      });
      
      // Convert to object structure for processTopProducts
      const demoProducts = {};
      productData.forEach(product => {
        // Add image URL
        product.image_url = this.determineProductImageUrl({ name: product.name }, product.name);
        demoProducts[product.name] = product;
      });
      
      // Process these products
      this.processTopProducts(demoProducts, currentMonth);
      
      // Generate monthly data
      this.monthlyData = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'].map((month, index) => {
        // Generate random sales data
        const sales = index === currentMonth ? 500000 : Math.random() * 400000;
        return {
          month,
          sales,
          daysWithData: 30,
          orderCount: Math.round(sales / 5000) // Rough estimate for demo
        };
      });
      
      // Calculate statistics
      this.calculateStatistics();
      this.preloadProductImages();
      
      // Set loading to false
      this.loading = false;
      
      // Safely initialize charts after data is loaded
      this.safeInitializeCharts();
      
      this.toast.success('Demo data loaded successfully');
    },
    
    // Safely initialize charts with proper timing
    safeInitializeCharts() {
      console.log('Scheduling safe chart initialization...');
      
      // Use nextTick to ensure Vue has updated the DOM
      nextTick(() => {
        // Use a longer timeout to ensure DOM is fully rendered
        setTimeout(() => {
          console.log('Rendering monthly sales chart...');
          this.renderMonthlySalesChart();
          
          // Render predictions chart after a short delay
          setTimeout(() => {
            console.log('Rendering predictions chart...');
            this.renderPredictionsChart();
          }, 500);
        }, 800);
      });
    },
    
    // Preload product images to avoid loading issues
    preloadProductImages() {
      if (!this.topProducts || this.topProducts.length === 0) return;
      
      console.log('Preloading product images...');
      
      this.topProducts.forEach(product => {
        if (product.image_url) {
          const img = new Image();
          img.onload = () => console.log(`Successfully preloaded image for ${product.name}`);
          img.onerror = () => {
            console.warn(`Failed to preload image for ${product.name}, generating fallback`);
            // Generate a fallback image using our helper
            product.fallback_image = this.createFallbackImage(product.name);
          };
          img.src = this.getFixedImageUrl(product);
        }
      });
    },
    
    // Helper method to create a fallback image
    createFallbackImage(productName) {
      try {
        const initial = (productName || '?').charAt(0).toUpperCase();
        const color = this.getHashColor(productName || 'default');
        
        // Create canvas element
        const canvas = document.createElement('canvas');
        canvas.width = 64;
        canvas.height = 64;
        
        // Get context - if this fails, return a default base64 image
        const ctx = canvas.getContext('2d');
        if (!ctx) {
          console.warn('Failed to get canvas context');
          return 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAMElEQVQ4jWNgGAXDAAxh8P//f1EM9/9UyZb//8hzAEwdTQ1AdjnlDsBnKM0CGQAAgaYemQNz0rAAAAAASUVORK5CYII=';
        }
        
        // Draw colored background
        ctx.fillStyle = color;
        ctx.fillRect(0, 0, 64, 64);
        
        // Draw text
        ctx.fillStyle = 'white';
        ctx.font = 'bold 28px Arial, Sans-serif';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(initial, 32, 32);
        
        // Return canvas data URL
        return canvas.toDataURL('image/png');
      } catch (e) {
        console.error('Error creating canvas fallback:', e);
        // Simple colored div as ultimate fallback
        return 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAMElEQVQ4jWNgGAXDAAxh8P//f1EM9/9UyZb//8hzAEwdTQ1AdjnlDsBnKM0CGQAAgaYemQNz0rAAAAAASUVORK5CYII=';
      }
    },
    
    // Toggle between logarithmic and linear scale
    toggleChartScale() {
      this.useLogScale = !this.useLogScale;
      this.renderMonthlySalesChart();
    },
    
    // Clean product names for better display
    cleanProductName(name) {
      if (!name) return 'Product';
      
      // Remove numeric IDs from the beginning (like 5435345 Geeyt → Geeyt)
      const nameWithoutId = name.replace(/^\d+\s+/, '');
      
      // If the name became empty after removing numbers, return the original
      if (!nameWithoutId.trim()) return name;
      
      // Capitalize the first letter of each word
      return nameWithoutId.split(' ').map(word => 
        word.charAt(0).toUpperCase() + word.slice(1).toLowerCase()
      ).join(' ');
    },
  }
};
</script>

<style scoped>
.app-container {
  padding: 20px;
  margin-left: 260px;
  margin-top: 80px; /* Add top margin to account for fixed header */
  transition: margin-left 0.3s;
  overflow-y: auto; /* Enable scrolling for the container */
  position: relative; /* Add relative positioning */
  height: calc(100vh - 80px); /* Set height to account for header */
}

.sales-container {
  max-width: 1400px;
  margin: 0 auto;
}

.header-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.section-title {
  font-size: 24px;
  color: #333;
  margin: 0;
}

.dashboard-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 20px;
  margin-bottom: 30px;
}

.card {
  background: white;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  padding: 20px;
}

.stats-card {
  text-align: center;
}

.stats-card h3 {
  font-size: 16px;
  color: #777;
  margin-bottom: 8px;
  font-weight: 500;
}

.stat-value {
  font-size: 26px;
  font-weight: 700;
  color: #333;
}

.charts-section {
  display: grid;
  grid-template-columns: 1fr;
  gap: 20px;
  margin-bottom: 30px;
}

.chart-card {
  min-height: 400px;
}

.chart-card h3 {
  font-size: 18px;
  color: #555;
  margin-bottom: 15px;
  font-weight: 500;
}

.chart-container {
  height: 350px;
  position: relative;
}

/* Responsive adjustments */
@media (max-width: 1024px) {
  .dashboard-row {
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  }
}

@media (max-width: 768px) {
  .app-container {
    margin-left: 0;
    padding: 15px;
    margin-top: 80px; /* Keep the top margin for header */
    height: calc(100vh - 80px); /* Maintain height for smaller screens */
  }
  
  .dashboard-row {
    grid-template-columns: 1fr;
  }
  
  .charts-section {
    grid-template-columns: 1fr;
  }
}

/* Add this new CSS for the top products table */
.top-products-section {
  margin-bottom: 30px;
  width: 100%;
  overflow: visible;
}

.top-products-card {
  min-height: auto;
  padding: 20px;
  background: white;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  overflow: hidden; /* Prevent content overflow */
}

.top-products-card h3 {
  font-size: 20px;
  color: #333;
  margin-top: 0;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid #eee;
  font-weight: 600;
}

.top-products-chart-container {
  padding: 10px 0;
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
  display: block;
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

/* Add this new CSS for the predictions section */
.predictions-section {
  margin-bottom: 30px;
}

.predictions-card {
  min-height: auto;
  padding: 20px;
  background: white;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}

.predictions-card h3 {
  font-size: 20px;
  color: #333;
  margin-top: 0;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid #eee;
  font-weight: 600;
}

.prediction-summary {
  margin-bottom: 20px;
}

.prediction-products {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.prediction-product {
  display: flex;
  align-items: center;
  gap: 10px;
}

.prediction-rank {
  font-size: 16px;
  font-weight: bold;
  color: #666;
  min-width: 36px;
  text-align: center;
}

.prediction-name {
  font-size: 16px;
  font-weight: 500;
  color: #333;
  text-align: left;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 300px;
}

.prediction-value {
  font-size: 16px;
  font-weight: 700;
  color: #333;
}

.prediction-info {
  text-align: right;
  padding: 8px 15px;
  color: #888;
  font-size: 0.8rem;
  border-top: 1px solid #eee;
  margin-top: 5px;
}

/* Add this at the top of your existing styles */
.refresh-button {
  cursor: pointer;
  font-size: 14px;
  margin-left: 10px;
  color: #888;
  vertical-align: middle;
  transition: color 0.3s;
}

.refresh-button:hover {
  color: #2c7be5;
}

/* Combined badge style */
.combined-badge {
  display: inline-block;
  font-size: 10px;
  font-weight: 500;
  background-color: #6b5eae;
  color: white;
  padding: 2px 6px;
  border-radius: 10px;
  margin-left: 8px;
  vertical-align: middle;
  text-transform: uppercase;
}

.unnamed-note {
  font-size: 11px;
  color: #999;
  font-style: italic;
  margin-left: 5px;
}

/* Add this to the <style> section */
.scale-toggle {
  font-size: 12px;
  font-weight: normal;
  color: #2c7be5;
  cursor: pointer;
  margin-left: 10px;
  padding: 2px 8px;
  border-radius: 12px;
  background-color: #f0f7ff;
  transition: all 0.2s ease;
  display: inline-block;
  vertical-align: middle;
}

.scale-toggle:hover {
  background-color: #e0efff;
  color: #1a68d4;
}
</style> 