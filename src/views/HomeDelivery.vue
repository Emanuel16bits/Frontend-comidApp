<template>
  <div class="home-container">
    
    <header class="vendor-header">
      <div class="header-content">
        <div class="logo-section">
          <h1>comidApp</h1>
        </div>

        <div class="user-section">
          <button class="btn-profile" @click="toggleProfileMenu">
            👤 {{ userName }}
          </button>

          
          <div v-if="showProfileMenu" class="profile-dropdown">
            <button @click="showVehicleForm = true; showProfileMenu = false" class="dropdown-item">
              <i class="fa-solid fa-car"></i> Registrar Vehículo
            </button>
            <button @click="fetchDriverVehicles" class="dropdown-item">
              <i class="fa-solid fa-list"></i> Ver Mi Vehículo
            </button>
            <div class="dropdown-divider"></div>
            <button @click="logout" class="dropdown-item logout-btn">
              <i class="fa-solid fa-right-from-bracket"></i> Cerrar sesión
            </button>
          </div>
        </div>
      </div>
    </header>
    
    <div v-if="showVehicleForm" class="modal-overlay" @click.self="showVehicleForm = false">
      <div class="modal-content">
        <button class="close-modal" @click="showVehicleForm = false">&times;</button>
        <VehicleForm 
          @saved="handleVehicleSaved"
          @cancel="showVehicleForm = false"
        />
      </div>
    </div>
    
    <div v-if="loading" class="loading">Cargando pedidos...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else-if="orders.length === 0" class="no-orders">
      No hay pedidos disponibles en este momento.
    </div>
    <div v-else class="orders-grid">
      <div v-for="order in orders" :key="order.id" class="order-card">
        <div class="order-header">
          <span class="order-id">#{{ order.id }}</span>
          <span class="order-status" :class="order.estado">
            {{ formatStatus(order.estado) }}
          </span>
        </div>

        <div class="order-details">
          <div class="order-customer">
            
            <div>
              <h4>Cliente: {{ order.usuario?.nombre || 'Cliente' }}</h4>
              <p>ID: #{{ order.idUsuario || 'N/A' }}</p>
            </div>
          </div>

          <div class="restaurant-info" v-if="order.restaurante && order.restaurante.nombre">
            
            <div>
              <h4>{{ order.restaurante.nombre }}</h4>
              <p>{{ order.restaurante.direccion || 'Dirección no disponible' }}</p>
            </div>
          </div>
          <div class="restaurant-info" v-else>
            
            <div>
              <h4>Restaurante no disponible</h4>
              <p>No se pudo cargar la información del restaurante</p>
            </div>
          </div>

          <div class="order-total">
            <strong>Total del pedido:</strong>
            <span>${{ typeof order.precioTotal === 'number' ? order.precioTotal.toFixed(2) : '0.00' }}</span>
          </div>

          <div class="order-actions">
            <button 
              v-if="order.estado === 'pendiente'" 
              @click="updateOrderStatus(order.id, 'en_camino')"
              :disabled="updatingOrderId === order.id"
              class="btn-accept"
            >
              <span v-if="updatingOrderId !== order.id">Aceptar pedido</span>
              <span v-else>Aceptando...</span>
            </button>
            
            <button 
              v-if="order.estado === 'en_camino'" 
              @click="updateOrderStatus(order.id, 'entregada')"
              :disabled="updatingOrderId === order.id"
              class="btn-delivered"
            >
              <span v-if="updatingOrderId !== order.id">Marcar como entregado</span>
              <span v-else>Actualizando...</span>
            </button>
          </div>
        </div>
      </div>
    </div>

    <div v-if="showVehiclesModal" class="modal-overlay" @click.self="showVehiclesModal = false">
      <div class="modal-content">
        <button class="close-modal" @click="showVehiclesModal = false">&times;</button>
        <h3>Mis Vehículos Registrados</h3>
        <div v-if="vehiclesLoading" class="loading">Cargando vehículos...</div>
        <div v-else-if="vehiclesError" class="error">{{ vehiclesError }}</div>
        <div v-else-if="!driverVehicles" class="no-vehicles">
          No tienes vehículos registrados.
        </div>
        <div v-else class="vehicle-details">
          <div class="vehicle-info">
            <p><strong>Tipo:</strong> {{ driverVehicles.tipoVehiculo || 'No especificado' }}</p>
            <p><strong>Marca:</strong> {{ driverVehicles.marca || 'No especificada' }}</p>
            <p><strong>Modelo:</strong> {{ driverVehicles.modelo || 'No especificado' }}</p>
            <p><strong>Color:</strong> {{ driverVehicles.color || 'No especificado' }}</p>
            <p><strong>Año:</strong> {{ driverVehicles.anio || 'No especificado' }}</p>
            <p><strong>Placa:</strong> {{ driverVehicles.placa || 'No especificada' }}</p>
          </div>
        </div>
      </div>
    </div>
  </div>

     <footer class="home-footer">
      <p>© 2025 comidApp — Todos los derechos reservados</p>
    </footer>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { useAuthStore } from '@/stores/auth';
import axios from 'axios';
import VehicleForm from '@/components/VehicleForm.vue';
import { useRouter } from 'vue-router'

const showVehicleForm = ref(false);
const showVehiclesModal = ref(false);
const driverVehicles = ref(null);
const vehiclesLoading = ref(false);
const vehiclesError = ref(null);
const router = useRouter()
const userName = ref('Usuario')
const showProfileMenu = ref(false)

const toggleProfileMenu = () => {
  showProfileMenu.value = !showProfileMenu.value
}

const logout = () => {
  localStorage.removeItem('user')
  localStorage.removeItem('token')
  router.push('/login')
}


const fetchDriverVehicles = async () => {
  try {
    vehiclesLoading.value = true;
    vehiclesError.value = null;
    showVehiclesModal.value = true;
    
    const userId = authStore.user?.id;
    if (!userId) {
      throw new Error('Usuario no autenticado');
    }

    const response = await axios.get(`${API_URL}/drivers/user/${userId}/vehicles`, {
      headers: {
        'Authorization': `Bearer ${authStore.token}`
      }
    });

    driverVehicles.value = response.data;
  } catch (error) {
    console.error('Error al obtener vehículos:', error);
    vehiclesError.value = 'No se pudieron cargar los vehículos. Intente de nuevo.';
  } finally {
    vehiclesLoading.value = false;
  }
};

const handleVehicleSaved = (vehicleData) => {
  showVehicleForm.value = false;
  alert('Vehículo registrado exitosamente');
};

const API_URL = 'http://localhost:3000';
const authStore = useAuthStore();
const orders = ref([]);
const loading = ref(true);
const error = ref(null);
const updatingOrderId = ref(null);

const formatStatus = (status) => {
  const statusMap = {
    'pendiente': 'Pendiente',
    'en_camino': 'En camino',
    'entregada': 'Entregada',
    'cancelada': 'Cancelada'
  };
  return statusMap[status] || status;
};

const fetchOrders = async () => {
  try {
    loading.value = true;
    error.value = null;
    
    const restaurantsResponse = await axios.get(`${API_URL}/restaurants`);
    const restaurants = restaurantsResponse.data;
    
    const ordersResponse = await axios.get(`${API_URL}/orders?_embed=orderItems&_expand=usuario`);
    
    const defaultRestaurant = restaurants.length > 0 ? restaurants[0] : null;
    
    const ordersWithDetails = ordersResponse.data.map(order => {
    const restaurante = defaultRestaurant || {};
      
      const precioTotal = typeof order.precioTotal === 'string' 
        ? parseFloat(order.precioTotal) 
        : order.precioTotal || 0;

      return {
        ...order,
        precioTotal,
        restaurante,
        usuario: order.usuario || { nombre: 'Cliente' }
      };
    });

    orders.value = ordersWithDetails.filter(order => 
      order.estado === 'pendiente' || 
      (order.estado === 'en_camino' && order.repartidorId === authStore.userId)
    );
    
  } catch (err) {
    console.error('Error al cargar pedidos:', err);
    error.value = 'Error al cargar los pedidos. Por favor, intente de nuevo.';
  } finally {
    loading.value = false;
  }
};

const updateOrderStatus = async (orderId, newStatus) => {
  try {
    updatingOrderId.value = orderId;
    
    const updateData = { estado: newStatus };
    
    if (newStatus === 'en_camino') {
      updateData.repartidorId = authStore.userId;
    }
    
    await axios.patch(`${API_URL}/orders/${orderId}`, updateData);
    
    const orderIndex = orders.value.findIndex(o => o.id === orderId);
    if (orderIndex !== -1) {
      orders.value[orderIndex].estado = newStatus;
      if (newStatus === 'en_camino') {
        orders.value[orderIndex].repartidorId = authStore.userId;
      }
    }
    
  } catch (err) {
    console.error('Error al actualizar el pedido:', err);
    error.value = 'Error al actualizar el pedido. Por favor, intente de nuevo.';
  } finally {
    updatingOrderId.value = null;
  }
};

onMounted(() => {
  if (authStore.isAuthenticated) {
    fetchOrders();
  }
 const user = JSON.parse(localStorage.getItem('user'))
  if (user?.nombre) {
    userName.value = user.nombre
  } else if (user?.username) {
    userName.value = user.username
  }
  
});
</script>

<style scoped>
.profile-dropdown {
  position: absolute;
  right: 1rem;
  top: 100%;
  background: white;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  min-width: 220px;
  z-index: 1000;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.dropdown-item {
  padding: 0.75rem 1.25rem;
  text-align: left;
  background: none;
  border: none;
  cursor: pointer;
  transition: background-color 0.2s;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: #333;
  font-size: 0.95rem;
}

.dropdown-item:hover {
  background-color: #f5f5f5;
}

.dropdown-item i {
  width: 20px;
  text-align: center;
}

.dropdown-divider {
  height: 1px;
  background-color: #e9ecef;
  margin: 0.25rem 0;
}

.logout-btn {
  color: #dc3545;
}

.logout-btn:hover {
  background-color: #f8d7da;
}

.btn-profile {
  position: relative;
}

.vendor-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 20px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  position: sticky;
  top: 0;
  z-index: 100;
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.logo-section h1 {
  color: white;
  font-size: 2rem;
  margin: 0;
  font-weight: bold;
}

.user-section {
  position: relative;
}

.btn-profile {
  background: rgba(255, 255, 255, 0.2);
  border: none;
  padding: 10px 20px;
  border-radius: 25px;
  cursor: pointer;
  color: white;
  backdrop-filter: blur(10px);
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 600;
  transition: all 0.3s;
}

.btn-profile:hover {
  background: rgba(255, 255, 255, 0.3);
}

.profile-dropdown {
   position: absolute;
  top: 100%;
  right: 0;
  margin-top: 10px;
  background: white;
  border-radius: 15px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
  min-width: 200px;
  overflow: hidden;
  animation: slideDown 0.3s ease;
}

.logout-btn {
  width: 100%;
  padding: 0.5rem 1rem;
  text-align: left;
  border: none;
  background: none;
  cursor: pointer;
  color: #333;
  font-weight: 500;
  transition: background 0.2s, color 0.2s;
}

.logout-btn:hover {
  background: rgba(0, 0, 0, 0.05);
  color: #000;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.orders-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
  margin-top: 20px;
}

.order-card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.order-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background-color: #f5f5f5;
  border-bottom: 1px solid #e0e0e0;
}

.order-id {
  font-weight: bold;
  color: #555;
}

.order-status {
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 0.85em;
  font-weight: 500;
}

.order-status.pendiente {
  background-color: #fff3cd;
  color: #856404;
}

.order-status.en_camino {
  background-color: #cce5ff;
  color: #004085;
}

.order-status.entregada {
  background-color: #d4edda;
  color: #155724;
}

.order-details {
  padding: 16px;
}

.order-customer,
.restaurant-info {
  display: flex;
  align-items: center;
  margin-bottom: 16px;
  padding: 12px;
  background-color: #f8f9fa;
  border-radius: 8px;
}

.order-customer .material-icons,
.restaurant-info .material-icons {
  margin-right: 12px;
  font-size: 24px;
  color: #4a6cf7;
}

.order-customer h4,
.restaurant-info h4 {
  margin: 0 0 4px;
  color: #212529;
}

.order-customer p,
.restaurant-info p {
  margin: 0;
  color: #6c757d;
  font-size: 0.9rem;
}

.order-total {
  display: flex;
  justify-content: space-between;
  padding: 12px 0;
  border-top: 1px solid #e9ecef;
  margin-top: 16px;
  font-size: 1.1em;
}

.order-total strong {
  color: #212529;
}

.order-total span {
  font-weight: bold;
  color: #28a745;
}

.order-actions {
  display: flex;
  justify-content: center; 
  margin-top: 1rem;    
}

.btn-accept {
  background-color: #4a6cf7;
  color: white;
}

.btn-accept:hover:not(:disabled) {
  background-color: #3a5bd9;
}

.btn-delivered {
  background-color: #28a745;
  color: white;
}

.btn-delivered:hover:not(:disabled) {
  background-color: #218838;
}

.loading,
.error,
.no-orders {
  text-align: center;
  padding: 40px;
  font-size: 1.1em;
  color: #6c757d;
}

.error {
  color: #dc3545;
}

@media (max-width: 768px) {
  .orders-grid {
    grid-template-columns: 1fr;
  }

  button {
    width: 100%;
  }
}
</style>
