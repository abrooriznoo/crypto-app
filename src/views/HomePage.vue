<template>
  <ion-page>

    <!-- ===== HEADER ===== -->
    <ion-header>
      <ion-toolbar class="header-main">

        <ion-title slot="start" class="header-left">
          💰 Crypto Market
        </ion-title>

        <ion-searchbar slot="end" v-model="search" class="header-search" placeholder="Search..."
          @ionInput="filterCoins" />

      </ion-toolbar>


      <!-- SORT -->
      <ion-toolbar class="sort-toolbar">
        <ion-segment v-model="sortBy" @ionChange="sortCoins">
          <ion-segment-button value="rank">
            <ion-label>Rank</ion-label>
          </ion-segment-button>
          <ion-segment-button value="price">
            <ion-label>Price</ion-label>
          </ion-segment-button>
          <ion-segment-button value="volume">
            <ion-label>Volume</ion-label>
          </ion-segment-button>
          <ion-segment-button value="favorite">
            <ion-label>⭐ Favorite</ion-label>
          </ion-segment-button>
        </ion-segment>
      </ion-toolbar>
    </ion-header>

    <!-- ===== CONTENT ===== -->
    <ion-content :fullscreen="true" class="crypto-content">

      <!-- PULL TO REFRESH -->
      <ion-refresher slot="fixed" @ionRefresh="handleRefresh">
        <ion-refresher-content pulling-text="Pull to refresh" refreshing-spinner="crescent" />
      </ion-refresher>

      <div class="content-wrapper">

        <!-- LOADING -->
        <div v-if="loading" class="loading-container">
          <ion-spinner name="crescent"></ion-spinner>
          <p class="loading-text">Loading market data...</p>
        </div>

        <!-- LIST -->
        <ion-list v-else class="coins-list">

          <ion-item lines="none" v-for="coin in coins" :key="coin.id" class="crypto-item">
            <ion-card class="crypto-card">

              <!-- HEADER -->
              <ion-card-header>
                <div class="card-header-content">

                  <div class="coin-info">
                    <div class="rank-badge">#{{ coin.rank }}</div>

                    <div>
                      <ion-card-title class="coin-name">
                        {{ coin.name }}
                      </ion-card-title>
                      <ion-card-subtitle class="coin-symbol">
                        {{ coin.symbol }}
                      </ion-card-subtitle>
                    </div>
                  </div>

                  <div class="right-box">

                    <!-- FAVORITE BUTTON -->
                    <ion-button fill="clear" class="fav-btn" @click="toggleFavorite(coin)">
                      <ion-icon :icon="isFavorite(coin.id) ? star : starOutline" />
                    </ion-button>

                    <!-- PERCENT -->
                    <div :class="[
                      'price-change',
                      getPriceChangeClass(coin.percent_change_24h),
                    ]">
                      {{ parseFloat(coin.percent_change_24h).toFixed(2) }}%
                    </div>
                  </div>

                </div>
              </ion-card-header>

              <!-- BODY -->
              <ion-card-content>

                <div class="price-section">
                  <div class="price-label">Current Price</div>
                  <div class="price-value">
                    ${{ formatPrice(coin.price_usd) }}
                  </div>
                </div>

                <div class="stats-grid">

                  <div class="stat-item">
                    <div class="stat-label">Market Cap</div>
                    <div class="stat-value">
                      ${{ formatMarketCap(coin.market_cap_usd) }}
                    </div>
                  </div>

                  <div class="stat-item">
                    <div class="stat-label">Volume 24h</div>
                    <div class="stat-value">
                      ${{ formatVolume(coin.volume24) }}
                    </div>
                  </div>

                </div>

              </ion-card-content>

            </ion-card>
          </ion-item>

        </ion-list>

        <!-- EMPTY -->
        <div v-if="!loading && coins.length === 0" class="loading-container">
          <p class="loading-text">No coins found</p>
        </div>

      </div>
    </ion-content>

  </ion-page>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import axios from "axios";

import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonCard,
  IonCardHeader,
  IonCardTitle,
  IonCardSubtitle,
  IonCardContent,
  IonSpinner,
  IonSearchbar,
  IonList,
  IonItem,
  IonButton,
  IonIcon,
  IonSegment,
  IonSegmentButton,
  IonLabel,
  IonRefresher,
  IonRefresherContent
} from "@ionic/vue";

import { star, starOutline } from "ionicons/icons";
import { toastController, onIonViewDidEnter } from '@ionic/vue'

// STATE
const coins = ref([]);
const allCoins = ref([]);
const loading = ref(true);
const search = ref("");
const sortBy = ref("rank");
const favorites = ref(JSON.parse(localStorage.getItem("favorites") || "[]"));
let interval = null;

// HELPER TOAST
const showToast = async (message) => {
  const toast = await toastController.create({
    message,
    duration: 3000,
    position: 'top', // WAJIB top
    cssClass: 'toastify-toast toastify-toast--success',
    animated: true,
    buttons: [
      {
        text: '✕',
        role: 'cancel'
      }
    ]
  })

  await toast.present()
}

const showToastError = async (message) => {
  const toast = await toastController.create({
    message,
    duration: 3000,
    position: 'top', // WAJIB top
    cssClass: 'toastify-toast toastify-toast--error',
    animated: true,
    buttons: [
      {
        text: '✕',
        role: 'cancel'
      }
    ]
  })

  await toast.present()
}


// GET DATA
const getCoins = async () => {
  try {
    const response = await axios.get("https://api.coinlore.net/api/tickers/");
    allCoins.value = response.data.data.slice(0, 30);
    filterCoins();
    showToast("Data refreshed");
  } catch (err) {
    console.error("API Error:", err);
    showToastError("Failed to fetch data");
  } finally {
    loading.value = false;
  }
};

// FILTER + SORT
const filterCoins = () => {
  let data = [...allCoins.value];

  if (search.value) {
    const key = search.value.toLowerCase();
    data = data.filter(
      (c) =>
        c.name.toLowerCase().includes(key) ||
        c.symbol.toLowerCase().includes(key)
    );
  }

  // SORT
  if (sortBy.value === "price") {
    data.sort((a, b) => parseFloat(b.price_usd) - parseFloat(a.price_usd));
  } else if (sortBy.value === "volume") {
    data.sort((a, b) => parseFloat(b.volume24) - parseFloat(a.volume24));
  } else if (sortBy.value === "favorite") {
    data = data.filter((c) => favorites.value.includes(c.id));
  } else {
    data.sort((a, b) => a.rank - b.rank);
  }

  coins.value = data;
};

const sortCoins = () => filterCoins();

// FAVORITE
const toggleFavorite = (coin) => {
  const index = favorites.value.indexOf(coin.id);

  if (index === -1) {
    favorites.value.push(coin.id);
  } else {
    favorites.value.splice(index, 1);
  }

  localStorage.setItem("favorites", JSON.stringify(favorites.value));
  filterCoins();
};

const isFavorite = (id) => {
  return favorites.value.includes(id);
};

// REFRESH
const handleRefresh = async (event) => {
  await getCoins();
  event.target.complete();
};

// FORMATTER
const formatPrice = (price) => {
  const num = parseFloat(price);
  if (isNaN(num)) return "N/A";
  return num < 1
    ? num.toFixed(6)
    : num.toLocaleString("en-US", {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
    });
};

const formatMarketCap = (value) => {
  const num = parseFloat(value);
  if (num >= 1e12) return (num / 1e12).toFixed(2) + " T";
  if (num >= 1e9) return (num / 1e9).toFixed(2) + " B";
  if (num >= 1e6) return (num / 1e6).toFixed(2) + " M";
  return num.toFixed(2);
};

const formatVolume = (value) => {
  const num = parseFloat(value);
  if (num >= 1e9) return (num / 1e9).toFixed(2) + " B";
  if (num >= 1e6) return (num / 1e6).toFixed(2) + " M";
  return num.toFixed(2);
};

const getPriceChangeClass = (change) => {
  const num = parseFloat(change);
  return num >= 0 ? "positive" : "negative";
};

// REALTIME
onMounted(() => {
  getCoins();
  interval = setInterval(getCoins, 15000); // Refresh every 15 seconds
});

onUnmounted(() => {
  if (interval) clearInterval(interval);
});
</script>

<style scoped>
.crypto-content {
  /* --background: #f0f4f8; */
  --background: #2e2e2e;
  color: #ffffff;
}

/* ========= WRAPPER ========= */
.content-wrapper {
  /* padding: 14px; */
  padding: 10px;
  color: #ffffff;
}

.header-main {
  --background: #2e2e2e;
  /* --background: #ffffff; */
  border-bottom: 1px solid #212121;
  padding-bottom: 2px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #ffffff;
}

.header-left {
  text-align: left;
  color: #ffffff;
  padding-left: 10px;
}

.header-search {
  color: #ffffff;
  margin-top: 15px;
  max-width: 150px;
}

/* Tablet & Desktop */
@media (min-width: 768px) {
  .header-search {
    max-width: 300px;
  }
}

/* ========= LIST -> GRID RESPONSIVE ========= */
.coins-list {
  display: grid;
  grid-template-columns: 1fr;
  color: #ffffff;
  gap: 14px;
}

/* Tablet: 2 kolom */
@media (min-width: 768px) {
  .coins-list {
    grid-template-columns: repeat(2, 1fr);
  }
}

/* Desktop: 3 kolom */
@media (min-width: 1200px) {
  .coins-list {
    grid-template-columns: repeat(3, 1fr);
  }
}

/* ========= ITEM ========= */
.crypto-item {
  --background: transparent;
  padding: 0;
  color: #ffffff;
  margin: 0;
}

/* ========= CARD ========= */
.crypto-card {
  width: 100%;
  border-radius: 18px;
  border: solid 1px rgb(93, 93, 93);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  margin: 0;
  color: #ffffff;
  box-shadow: 0 6px 14px rgba(0, 0, 0, 0.05);
  margin-top: 15px;
  margin-bottom: 15px;
}

@media (hover: hover) {
  .crypto-card:hover {
    transform: translateY(-4px);
    box-shadow: 0 12px 24px rgba(0, 0, 0, 0.15);
    transition: all 0.3s ease;
  }
}

@media (max-width: 480px) {
  .crypto-card {
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    margin-bottom: 15px;
  }
}

/* ========= HEADER ========= */
.card-header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #ffffff;
  flex-wrap: nowrap;
}

.coin-info {
  display: flex;
  align-items: center;
  gap: 10px;
  color: #ffffff;
}

/* Mobile: lebih kecil */
@media (max-width: 480px) {
  .coin-info {
    gap: 6px;
  }
}

.rank-badge {
  background: #6366f1;
  color: white;
  padding: 5px 8px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: bold;
}

/* ========= TEXT ========= */
.coin-name {
  font-size: 16px;
  font-weight: 700;
  color: #ffffff;
}

.coin-symbol {
  font-size: 13px;
}

/* Mobile kecil */
@media (max-width: 480px) {
  .coin-name {
    font-size: 14px;
  }

  .coin-symbol {
    font-size: 12px;
  }
}

.right-box {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #ffffff;
}

/* ========= FAVORITE ========= */
.fav-btn {
  --color: #facc15;
  font-size: 18px;
}

/* ========= CHANGE ========= */
.price-change {
  padding: 5px 8px;
  border-radius: 10px;
  font-weight: bold;
  font-size: 13px;
  min-width: 65px;
  text-align: center;
  color: #ffffff;
}

.positive {
  background: #dcfce7;
  color: #166534;
}

.negative {
  background: #fee2e2;
  color: #7f1d1d;
}

/* ========= PRICE ========= */
.price-section {
  margin: 8px 0;
  padding: 10px;
  border-radius: 14px;
  background: #454545;
  /* --background: #757575; */
  color: #ffffff;
}

.price-label {
  font-size: 12px;
  color: #ffffff;
}

.price-value {
  font-size: 18px;
  font-weight: 800;
  color: #ffffff;
}

/* Desktop font lebih besar */
@media (min-width: 1200px) {
  .price-value {
    font-size: 22px;
  }
}

/* ========= STATS ========= */
.stats-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.stat-label {
  font-size: 11px;
  color: #666;
}

.stat-value {
  font-weight: 600;
  font-size: 14px;
}

/* ========= LOADING ========= */
.loading-container {
  min-height: 60vh;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
}

/* ========= SORT TOOLBAR ========= */
.sort-toolbar {
  /* --background: #ffffff; */
  --background: #2e2e2e;
  padding: 5px 6px;
}

.sort-label {
  color: #ffffff;
}
</style>
