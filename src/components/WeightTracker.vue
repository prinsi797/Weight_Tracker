<template>
  <div class="weight-tracker">
    <!-- Header with Registration Button -->
    <div class="header">
      <h1>Weight Tracker</h1>
      <button v-if="!user" @click="showRegisterForm = true" class="register-btn">Register</button>
      <div v-else class="user-info">
        <span>Welcome, {{ user.name }}</span>
        <button @click="logout" class="logout-btn">Logout</button>
      </div>
    </div>

    <!-- Registration Form Modal -->
    <div v-if="showRegisterForm" class="modal-overlay">
      <div class="modal-content">
        <h2>Register</h2>
        <form @submit.prevent="registerUser">
          <div class="form-group">
            <label for="name">Name</label>
            <input type="text" id="name" v-model="registerForm.name" required>
          </div>
          <div class="form-group">
            <label for="email">Email</label>
            <input type="email" id="email" v-model="registerForm.email" required>
          </div>
          <div class="form-group">
            <label for="password">Password</label>
            <input type="password" id="password" v-model="registerForm.password" required>
          </div>
          <div class="form-actions">
            <button type="button" @click="showRegisterForm = false">Cancel</button>
            <button type="submit">Register</button>
          </div>
        </form>
      </div>
    </div>
    
    <!-- Current Weight Circle Display -->
    <div class="weight-circle">
      <div class="weight-circle-inner">
        <div class="weight-value">{{ currentWeight }}</div>
        <div class="weight-label">Current Weight (kg)</div>
      </div>
    </div>
    
    <!-- Weight Input Form -->
    <div class="weight-input">
      <input 
        type="number" 
        v-model="newWeight" 
        step="0.1" 
        placeholder="Enter weight"
        class="weight-input-field"
      />
      <button @click="addWeight" class="add-weight-btn">Add weight</button>
    </div>
    
    <!-- Weekly Report Chart -->
    <div class="chart-section">
      <h2>Weekly Report</h2>
      <div class="weight-chart">
        <canvas ref="weightChart"></canvas>
      </div>
    </div>
    
    <!-- Email Report Preview -->
    <div v-if="user" class="email-report-section">
      <h2>Weekly Email Report</h2>
      <p>Next report will be sent to {{ user.email }} on {{ nextReportDate }}</p>
      <button @click="previewEmailReport" class="preview-report-btn">Preview Report</button>
    </div>

    <!-- Email Preview Modal -->
    <div v-if="showEmailPreview" class="modal-overlay">
      <div class="modal-content email-preview">
        <h2>Weekly Weight Report</h2>
        <p>Hello {{ user ? user.name : '' }},</p>
        <p>Here is your weight tracking data for the past week:</p>
        
        <table class="report-table">
          <thead>
            <tr>
              <th>Date</th>
              <th>Weight (kg)</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(entry, index) in weeklyReportData" :key="index">
              <td>{{ formatDate(entry.date) }}</td>
              <td>{{ entry.weight }} kg</td>
            </tr>
          </tbody>
        </table>
        
        <p>Keep up the good work!</p>
        <p>Weight Tracker App</p>
        
        <div class="modal-actions">
          <button class="close-preview" @click="showEmailPreview = false">Close Preview</button>
        </div>
      </div>
    </div>
    
    <!-- Weight History -->
    <div class="history-section">
      <h2>Weight History</h2>
      <div class="weight-history">
        <div v-if="weightHistory.length === 0" class="no-data">
          No weight entries yet. Add your first weight above.
        </div>
        <div v-for="(entry, index) in weightHistory" :key="index" class="history-item">
          <div class="history-weight">{{ entry.weight }}kg</div>
          <div class="history-date">{{ formatDate(entry.date) }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Chart from 'chart.js/auto'

export default {
  name: 'WeightTracker',
  data() {
    return {
      newWeight: '',
      weightHistory: [],
      chart: null,
      user: null,
      showRegisterForm: false,
      registerForm: {
        name: '',
        email: '',
        password: ''
      },
      showEmailPreview: false,
      lastReportDate: null
    }
  },
  computed: {
    currentWeight() {
      return this.weightHistory.length > 0 ? this.weightHistory[0].weight : 0
    },
    weeklyReportData() {
      // Get data for the weekly report (last 7 days)
      const oneWeekAgo = new Date();
      oneWeekAgo.setDate(oneWeekAgo.getDate() - 7);
      
      return this.weightHistory.filter(entry => {
        const entryDate = new Date(entry.date);
        return entryDate >= oneWeekAgo;
      });
    },
    nextReportDate() {
      // Calculate when the next weekly report will be sent
      const nextDate = new Date();
      nextDate.setDate(nextDate.getDate() + (7 - nextDate.getDay()));
      return this.formatDate(nextDate);
    }
  },
  mounted() {
    this.loadData();
    this.createChart();
    this.setupWeeklyEmailCheck();
  },
  methods: {
    getChartData() {
      // Move chart data generation to a method to avoid recursion issues
      if (this.weightHistory.length === 0) {
        return {
          labels: [this.formatDate(new Date())],
          data: [0]
        };
      }
      
      // Create date for initial zero point to match first entry date
      const initialDate = new Date(this.weightHistory[this.weightHistory.length - 1].date);
      
      // Create data points with initial zero and actual entries
      const labels = [this.formatDate(initialDate)];
      const data = [0];
      
      // Add actual weight entries
      this.weightHistory.forEach(entry => {
        labels.push(this.formatDate(entry.date));
        data.push(entry.weight);
      });
      
      return { labels, data };
    },
    addWeight() {
  if (!this.newWeight) return;

  const weight = parseFloat(this.newWeight);
  if (isNaN(weight) || weight <= 0) {
    alert('Please enter a valid weight');
    return;
  }

  this.weightHistory.unshift({
    weight,
    date: new Date()
  });

  this.newWeight = '';
  this.saveData();

  // Delay chart update until DOM updates
  this.$nextTick(() => {
    this.updateChart();
  });
},
    formatDate(date) {
      if (!date) return '';
      const d = new Date(date);
      return d.toLocaleDateString('en-GB', {
        day: '2-digit',
        month: '2-digit',
        year: '2-digit'
      });
    },
    createChart() {
      if (!this.$refs.weightChart) return;
      
      const ctx = this.$refs.weightChart.getContext('2d');
      const chartData = this.getChartData();
      
      this.chart = new Chart(ctx, {
        type: 'line',
        data: {
          labels: chartData.labels,
          datasets: [{
            label: 'Weight',
            data: chartData.data,
            borderColor: '#ff69b4',
            backgroundColor: 'rgba(255, 105, 180, 0.2)',
            tension: 0.3,
            fill: true
          }]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          scales: {
            y: {
              beginAtZero: true
            }
          }
        }
      });
    },
    updateChart() {
     if (!this.chart) {
  this.createChart();
  return;
}

      
      const chartData = this.getChartData();
      
      this.chart.data.labels = chartData.labels;
      this.chart.data.datasets[0].data = chartData.data;
      
      // Update y-axis scale if needed
     // const maxWeight = Math.max(...chartData.data);
     // this.chart.options.scales.y.max = maxWeight + 5;
     const maxWeight = Math.max(...chartData.data.filter(n => typeof n === 'number'));
if (!isNaN(maxWeight)) {
  this.chart.options.scales.y.max = maxWeight + 5;
}

      
      this.chart.update();
    },
    saveData() {
  if (this.user) {
    this.user.weights = this.weightHistory;
    localStorage.setItem('user', JSON.stringify(this.user));
  }
},

    loadData() {
  const savedUser = localStorage.getItem('user');
  if (savedUser) {
    try {
      const parsedUser = JSON.parse(savedUser);
      this.user = parsedUser;
      this.weightHistory = (parsedUser.weights || []).map(entry => ({
        weight: entry.weight,
        date: new Date(entry.date)
      }));
    } catch (e) {
      console.error('Error loading saved user data', e);
      this.user = null;
      this.weightHistory = [];
    }
  }
},
    registerUser() {
  this.user = { 
    name: this.registerForm.name,
    email: this.registerForm.email,
    password: this.registerForm.password,
    weights: []
  };

  localStorage.setItem('user', JSON.stringify(this.user));

  this.weightHistory = []; // Reset weights for new user

  this.showRegisterForm = false;
  this.registerForm = { name: '', email: '', password: '' };
  alert('Registration successful!');
},
    logout() {
      // Remove user from localStorage
      this.user = null;
      localStorage.removeItem('user');
    },
    previewEmailReport() {
      this.showEmailPreview = true;
    },
    setupWeeklyEmailCheck() {
      // Check if it's time to send weekly report (simulated)
      setInterval(() => {
        if (!this.user) return;
        
        const now = new Date();
        const nextReportDate = new Date(this.lastReportDate || now);
        nextReportDate.setDate(nextReportDate.getDate() + 7);
        
        if (now >= nextReportDate) {
          this.sendWeeklyReport();
          this.lastReportDate = now;
          localStorage.setItem('lastReportDate', now.toISOString());
        }
      }, 60000); // Check every minute (in a real app, this would be different)
    },
    sendWeeklyReport() {
      // In a real app, this would connect to an email service
      // For this demo, we'll just show a notification
      if (this.user && this.weeklyReportData.length > 0) {
        console.log(`Weekly report would be sent to ${this.user.email}`);
        alert(`Weekly weight report sent to ${this.user.email}`);
      }
    }
  }
}
</script>

<style scoped>
.weight-tracker {
  max-width: 450px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

.close-preview{
    background-color: #005580;
    color: white;
    padding: 10px;
    border-radius: 10px;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

h1 {
  margin: 0;
  font-size: 25px;
}

.register-btn, .logout-btn {
  background-color: #005580;
  color: white;
  border: none;
  padding: 8px 12px;
  border-radius: 4px;
  cursor: pointer;
}

.logout-btn {
  background-color: #005580;
  margin-left: 10px;
}

.user-info {
  display: flex;
  align-items: center;
}

.modal-overlay {
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

.modal-content {
  background-color: white;
  padding: 20px;
  border-radius: 8px;
  width: 80%;
  max-width: 500px;
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
}

.form-group input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 15px;
}

.form-actions button {
  padding: 8px 12px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.form-actions button[type="button"] {
  background-color: #f1f1f1;
}

.form-actions button[type="submit"] {
  background-color: #005580;
  color: white;
}

h2 {
  color: #666;
  font-size: 18px;
  margin: 10px 0;
}

.weight-circle {
  width: 200px;
  height: 200px;
  border-radius: 50%;
  margin: 0 auto 20px;
  border: 10px solid #005580;
  display: flex;
  align-items: center;
  justify-content: center;
}

.weight-circle-inner {
  text-align: center;
}

.weight-value {
  font-size: 40px;
  font-weight: bold;
}

.weight-label {
  font-size: 14px;
  color: #666;
}

.weight-input {
  display: flex;
  margin-bottom: 20px;
}

.weight-input-field {
  flex: 1;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 4px 0 0 4px;
  font-size: 16px;
}

.add-weight-btn {
  padding: 12px 15px;
  background-color: #005580;
  color: white;
  border: none;
  border-radius: 0 4px 4px 0;
  cursor: pointer;
  font-size: 16px;
}

.chart-section {
  margin-bottom: 20px;
}

.weight-chart {
  height: 200px;
  border: 1px solid #eee;
  padding: 10px;
  border-radius: 4px;
}

.email-report-section {
  margin: 20px 0;
  padding: 15px;
  background-color: #f9f9f9;
  border-radius: 4px;
  border: 1px solid #eee;
}

.preview-report-btn {
  background-color: #005580;
  color: white;
  border: none;
  padding: 8px 12px;
  border-radius: 4px;
  cursor: pointer;
  margin-top: 10px;
}

.email-preview {
  max-width: 600px;
}

.report-table {
  width: 100%;
  border-collapse: collapse;
  margin: 15px 0;
}

.report-table th, .report-table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.report-table th {
  background-color: #f2f2f2;
}

.modal-actions {
  margin-top: 20px;
  text-align: right;
}

.history-section {
  margin-top: 10px;
}

.history-item {
  display: flex;
  justify-content: space-between;
  padding: 15px 10px;
  border-bottom: 1px solid #eee;
  background-color: #f9f9f9;
  margin-bottom: 5px;
}

.history-weight {
  font-weight: bold;
}

.history-date {
  color: #666;
}

.no-data {
  text-align: center;
  padding: 20px;
  color: #666;
  font-style: italic;
}
</style>