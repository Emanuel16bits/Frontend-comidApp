<template>
  <div class="home-container">
 <header class="vendor-header">
      <div class="header-content">
        <div class="logo-section">
          <h1>comidApp</h1>
        </div>
        <div class="user-section">
          <button class="btn-profile" @click="showProfileMenu = !showProfileMenu">
            👤 {{ userName }}
          </button>
          <div v-if="showProfileMenu" class="profile-dropdown">
            <button @click="logout" class="logout-btn">Cerrar sesión</button>
          </div>
        </div>
      </div>
    </header>

    <main class="home-main">
      <h2>Panel del Vendedor</h2>
      <p>Gestiona tus restaurantes y productos desde aquí.</p>

      <div class="add-restaurant-container">
        <button class="add-restaurant-btn" @click="goTo('/registrar-restaurante')">
          <i class="fa-solid fa-plus"></i> Agregar Restaurante
        </button>

        <button class="action-btn" @click="goTo('/mis-productos')">
           Agregar Productos
        </button>
      </div>

      <section class="restaurants-section">
        <div class="section-header">
          <h2>Mis Restaurantes</h2>
        </div>
    

        <div v-if="loading" class="empty-state">
          <p>Cargando restaurantes...</p>
        </div>

        <div v-else-if="restaurants.length === 0" class="empty-state">
          <p>No tienes restaurantes registrados</p>
        </div>

        <div v-else class="restaurants-grid">
          <div v-for="restaurant in restaurants" :key="restaurant.id" class="restaurant-card">
            <div class="restaurant-image">
              <img 
                :src="getImageUrl(restaurant.imagenUrl || restaurant.imagen)" 
                :alt="restaurant.nombre" 
                @error="handleImageError"
                style="width: 100%; height: 180px; object-fit: cover;"
              >
            </div>
            <div class="restaurant-info">
              <h3>{{ restaurant.nombre }}</h3>
              <p class="restaurant-category">{{ restaurant.categoria }}</p>
              <p class="restaurant-address">{{ restaurant.direccion }}</p>
              <p class="restaurant-schedule">
                <i class="fa-regular fa-clock"></i> 
                {{ formatTime(restaurant.horarioApertura) }} - {{ formatTime(restaurant.horarioCierre) }}
              </p>
              
              <div class="restaurant-actions">
                <button class="btn-edit" @click.stop="editRestaurant(restaurant.id)">
                  <i class="fa-regular fa-pen-to-square"></i> Editar
                </button>
                <button class="btn-delete" @click.stop="deleteRestaurant(restaurant.id)">
                  <i class="fa-regular fa-trash"></i> Eliminar
                </button>
              </div>
              
             
            </div>
          </div>
        </div>
      </section>
    </main>
       <footer class="home-footer">
      <p>© 2025 comidApp — Todos los derechos reservados</p>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import { useRouter } from 'vue-router'
import { useAuthStore } from '@/stores/auth'
import axios from 'axios'
import { getProductsByUser } from '../services/productsService'

const router = useRouter()
const authStore = useAuthStore()
const restaurants = ref([])
const products = ref([])
const loading = ref(false)
const error = ref(null)
const selectedRestaurantId = ref(null)


const showProfileMenu = ref(false)
const userName = ref('Usuario')

const logout = () => {
  authStore.logout()
  router.push('/login')
}

const goTo = (path) => router.push(path)

const formatTime = (timeString) => {
  if (!timeString) return ''
  return timeString.slice(0, 5)
}

const getImageUrl = (imagePath) => {
  if (!imagePath) return 'https://placehold.co/300x180?text=Sin+imagen'
  if (imagePath.startsWith('http')) return imagePath
  if (imagePath.startsWith('/uploads/')) return `http://localhost:3000${imagePath}`
  return `http://localhost:3000/uploads/${imagePath}`
}

const handleImageError = (event) => {
  event.target.src = 'https://placehold.co/300x180?text=Error+cargando+imagen'
}

const fetchRestaurants = async () => {
  try {
    loading.value = true
    const user = JSON.parse(localStorage.getItem('user') || '{}')
    const userId = user?.id
    if (!userId) throw new Error('Usuario no autenticado')

    const res = await fetch('http://localhost:3000/restaurants')
    const all = await res.json()
    restaurants.value = all.filter(r => r.idUsuario == userId)
    if (restaurants.value.length) selectedRestaurantId.value = restaurants.value[0].id
  } catch (err) {
    console.error(err)
    error.value = err.message
  } finally {
    loading.value = false
  }
}

const fetchProducts = async () => {
  try {
    const user = JSON.parse(localStorage.getItem('user') || '{}')
    if (!user?.id) return
    const data = await getProductsByUser(user.id)
    products.value = Array.isArray(data) ? data : []
  } catch (error) {
    console.error('Error al cargar productos:', error)
  }
}

const editRestaurant = (id) => router.push(`/editar-restaurante/${id}`)
const deleteRestaurant = async (id) => {
  if (!confirm('¿Querés eliminar este restaurante?')) return
  await axios.delete(`http://localhost:3000/restaurants/${id}`)
  await fetchRestaurants()
}

onMounted(async () => {
  if (!authStore.token) return router.push('/login')

  const user = JSON.parse(localStorage.getItem('user'))
  userName.value = user?.nombre || user?.username || 'Usuario'

  document.addEventListener('click', (e) => {
    if (!e.target.closest('.user-section')) showProfileMenu.value = false
  })

  await fetchRestaurants()
  if (restaurants.value.length) await fetchProducts()
})

watch(selectedRestaurantId, async (id) => {
  if (id) await fetchProducts()
})
</script>

<style scoped>

.add-restaurant-container {
  display: flex;
  justify-content: flex-end; 
  align-items: center;
  gap: 1rem; 
  margin-bottom: 2rem; 
  width: 100%;
}

.add-restaurant-btn,
.action-btn {
  background: #764ba2;
  color: white;
  border: none;
  border-radius: 9999px;
  padding: 0.75rem 1.5rem;
  transition: background 0.3s;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-weight: 600;
}

.add-restaurant-btn:hover,
.action-btn:hover {
  background: #764ba2;
}
.vendor-header{
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
.profile-dropdown button,
.profile-dropdown a {
  color: black;
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

.logo-section h1 {
  color: white;
  font-size: 2rem;
  margin: 0;
  font-weight: bold;
}

.top-buttons {
  display: flex;
  justify-content: center;
  gap: 1rem;
  padding: 1rem;
  color:#764ba2;
}


.buttons-container {
  display: flex;
  justify-content: flex-end; 
  gap: 1rem;
  margin-bottom: 2rem; 
  margin-left: 1rem;
}

.product-card {
  background: white;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
  border: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  height: 100%;
  font-size: 0.8rem;
}

.product-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 10px 15px rgba(0, 0, 0, 0.1);
}

.product-image {
  height: 80px;
  overflow: hidden;
  background: #f8fafc;
  border-bottom: 1px solid #f1f5f9;
}

.product-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.product-info {
  padding: 1rem;
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.product-name {
  margin: 0 0 0.5rem 0;
  color: #1e293b;
  font-size: 1rem;
  font-weight: 600;
  line-height: 1.3;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.product-details {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.price-stock {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.price {
  font-weight: 700;
  color: #2563eb;
  font-size: 1rem;
  white-space: nowrap;
}

.category {
  font-size: 0.7rem;
  color: white;
  background: #4f46e5;
  padding: 0.2rem 0.5rem;
  border-radius: 8px;
  font-weight: 500;
  text-transform: capitalize;
}

.product-description {
  color: #64748b;
  margin: 0;
  font-size: 0.85rem;
  line-height: 1.4;
  display: -webkit-box;
  -webkit-box-orient: vertical;
  overflow: hidden;
  text-overflow: ellipsis;
}

@media (max-width: 768px) {
  .products-grid {
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 0.75rem;
  }
  
  .product-image {
    height: 120px;
  }
  
  .product-info {
    padding: 0.75rem;
  }
}

@media (max-width: 480px) {
  .products-grid {
    grid-template-columns: repeat(2, 1fr);
    gap: 0.5rem;
  }
  
  .product-image {
    height: 100px;
  }
  
  .product-name {
    font-size: 0.9rem;
  }
  
  .price {
    font-size: 0.9rem;
  }
  
  .product-description {
    font-size: 0.8rem;
  }
  
  .category {
    font-size: 0.6rem;
    padding: 0.15rem 0.4rem;
  }
}
.home-container {
  min-height: 100vh;
  background: #f5f5f5;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.home-header {
  background: #667eea;
  color: white;
  padding: 1rem 2rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.home-header h1 {
  margin: 0;
  font-size: 1.8rem;
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
  gap: 1rem;
}

.home-header .btn {
  padding: 10px 20px;
  border: none;
  border-radius: 15px;
  cursor: pointer;
  background: white;
  color: #667eea;
  font-weight: bold;
  transition: all 0.3s ease;
}

.home-header .btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.home-main {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.home-main h2 {
  color: #2d3748;
  margin-bottom: 0.5rem;
}

.home-main > p {
  color: #718096;
  margin-bottom: 2rem;
}

.home-options {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1.5rem;
  margin-bottom: 3rem;
}

.option-card {
  background: white;
  border-radius: 15px;
  padding: 1.5rem;
  text-align: center;
  cursor: pointer;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  border: 1px solid #e2e8f0;
}

.option-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.1);
}

.option-card i {
  font-size: 2rem;
  color: #667eea;
  margin-bottom: 1rem;
}

.option-card h3 {
  margin: 0.5rem 0;
  color: #2d3748;
}

.option-card p {
  margin: 0;
  color: #718096;
  font-size: 0.9rem;
}


.restaurants-section, .products-section {
  margin-top: 2rem;
  padding: 1.5rem;
  background: white;
  border-radius: 15px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.section-header h2 {
  margin: 0;
  color: #2d3748;
}

.restaurants-grid, .products-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.5rem;
}

.restaurant-card, .product-card {
  background: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  border: 1px solid #e2e8f0;
}

.restaurant-card:hover, .product-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 10px 15px rgba(0, 0, 0, 0.1);
}

.restaurant-image img, .product-card img {
  width: 100%;
  height: 180px;
  object-fit: cover;
  border-bottom: 1px solid #e2e8f0;
}

.restaurant-info, .product-info {
  padding: 1.25rem;
  position: relative;
}

.restaurant-info h3, .product-info h3 {
  margin: 0 0 0.5rem;
  color: #2d3748;
  font-size: 1.25rem;
}

.restaurant-category {
  display: inline-block;
  background: #edf2f7;
  color: #4a5568;
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.85rem;
  margin-bottom: 0.75rem;
}

.restaurant-address, .product-description {
  color: #4a5568;
  margin: 0.5rem 0;
  font-size: 0.95rem;
  line-height: 1.5;
}

.restaurant-schedule {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: #4a5568;
  font-size: 0.9rem;
  margin: 0.5rem 0;
}
.restaurant-actions {
  display: flex;
  gap: 0.5rem;
  margin: 1rem 0;
}


.btn-edit, .btn-delete {
  flex: 1;
  padding: 0.5rem;
  border: none;
  border-radius: 8px;
  font-size: 0.80rem;
  cursor: pointer;
  
  align-items: center;
  justify-content: center;
  gap: 0.4rem;
  transition: all 0.2s ease;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
  display: inline-flex;
  line-height: 1.2;
}

 

.btn-edit {
  background-color: #bee3f8;
  transform: translateY(-1px);
  color: #3182ce;
}

.btn-delete {
  background-color: #fed7d7;
  transform: translateY(-1px);
  color: #e53e3e;
}

.restaurant-status {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.85rem;
  font-weight: 500;
  margin-top: 0.5rem;
}

.restaurant-status.active {
  background: #c6f6d5;
  color: #22543d;
}

.restaurant-status:not(.active) {
  background: #fed7d7;
  color: #822727;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #718096;
  font-size: 1.1rem;
  background: #f8fafc;
  border-radius: 12px;
  border: 1px dashed #cbd5e0;
}

.product-price {
  font-weight: bold;
  color: #2f855a;
  margin: 0.5rem 0;
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

.product-description {
  color: #4a5568;
  font-size: 0.9rem;
  margin: 0.5rem 0 0;
}
@media (max-width: 768px) {
  .home-options,
  .restaurants-grid,
  .products-grid {
    grid-template-columns: 1fr;
  }
  
  .home-main {
    padding: 1rem;
  }
  
  .restaurants-section,
  .products-section {
    padding: 1rem;
  }


}
</style>