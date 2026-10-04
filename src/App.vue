<script setup>
import { onBeforeUnmount, ref } from 'vue'
import MovieCard from './components/MovieCard.vue'
import { fetchMovies } from './api.js'
import tmdbLogo from './assets/tmdb-logo.svg'

const draftKey = ref('')
const apiKey = ref('')

const query = ref('')
const activeQuery = ref('')

const movies = ref([])
const favorites = ref([])
const showFavorites = ref(false)

const page = ref(1)
const totalPages = ref(0)
const totalResults = ref(0)

const loading = ref(false)
const error = ref('')

let controller
let requestId = 0

async function loadMovies(nextPage = 1, term = activeQuery.value) {
  const id = ++requestId

  controller?.abort()

  const currentController = new AbortController()
  controller = currentController

  let timedOut = false

  const timer = setTimeout(() => {
    timedOut = true
    currentController.abort()
  }, 15000)

  loading.value = true
  error.value = ''
  movies.value = []

  try {
    const data = await fetchMovies({
      apiKey: apiKey.value,
      query: term,
      page: nextPage,
      signal: currentController.signal
    })

    if (id !== requestId) return

    movies.value = data.results
    page.value = data.page
    totalPages.value = Math.min(data.total_pages, 500)
    totalResults.value = data.total_results
    activeQuery.value = term
  } catch (err) {
    if (id !== requestId) return

    error.value = timedOut
      ? 'Request timed out. Check your connection and retry.'
      : err.name === 'AbortError'
        ? ''
        : err.status !== undefined
          ? err.message
          : 'Could not reach TMDB. Check your connection or try again later.'
  } finally {
    clearTimeout(timer)

    if (id === requestId) {
      loading.value = false
    }
  }
}

function connect() {
  const key = draftKey.value.trim()

  if (!key) return

  apiKey.value = key
  draftKey.value = ''
  activeQuery.value = ''
  query.value = ''
  showFavorites.value = false

  loadMovies(1, '')
}

function search() {
  showFavorites.value = false
  loadMovies(1, query.value.trim())
}

function popular() {
  query.value = ''
  showFavorites.value = false
  loadMovies(1, '')
}

function disconnect() {
  requestId++
  controller?.abort()

  apiKey.value = ''
  draftKey.value = ''
  query.value = ''
  activeQuery.value = ''

  movies.value = []
  favorites.value = []

  error.value = ''
  loading.value = false

  page.value = 1
  totalPages.value = 0
  totalResults.value = 0

  showFavorites.value = false
}

function clearSearch() {
  query.value = ''

  if (activeQuery.value) {
    showFavorites.value = false
    loadMovies(1, '')
  }
}

function toggleFavorite(movie) {
  const exists = favorites.value.some(
    item => item.id === movie.id
  )

  if (exists) {
    favorites.value = favorites.value.filter(
      item => item.id !== movie.id
    )
  } else {
    favorites.value.push(movie)
  }
}

function isFavorite(movie) {
  return favorites.value.some(
    item => item.id === movie.id
  )
}

function surpriseMe() {
  if (!movies.value.length) return

  const movie =
    movies.value[
      Math.floor(Math.random() * movies.value.length)
    ]

  window.open(
    `https://www.themoviedb.org/movie/${movie.id}`,
    '_blank',
    'noopener,noreferrer'
  )
}

onBeforeUnmount(() => {
  requestId++
  controller?.abort()
})
</script>

<template>
  <div class="app-shell">

    <!-- NAVIGATION -->
    <header class="site-header">
      <a class="brand" href="#top">
        <span class="brand-mark">M</span>

        <span>
          MOVIE<span class="brand-accent">LAB</span>
        </span>
      </a>

      <nav aria-label="Main navigation">
        <a href="#discover">Discover</a>
        <a href="#movies">Movies</a>
        <a href="#about">About</a>
      </nav>

      <div class="header-status">
        <span class="status-dot"></span>
        TMDB LIVE
      </div>
    </header>

    <main id="top">

      <!-- HERO -->
      <section
        id="discover"
        class="hero"
        aria-labelledby="hero-title"
      >
        <div class="hero-content">

          <div class="hero-copy">

            <p class="eyebrow">
              <span>✦</span>
              YOUR PERSONAL MOVIE DISCOVERY
            </p>

            <h1 id="hero-title">
              Find something
              <em>worth watching.</em>
            </h1>

            <p class="hero-description">
              Search thousands of movies, explore what's popular,
              and discover your next favourite story.
            </p>

            <!-- API KEY -->
            <form
              v-if="!apiKey"
              class="key-form"
              @submit.prevent="connect"
            >
              <label for="tmdb-key">
                Connect your TMDB account
              </label>

              <div class="search-field">
                <input
                  id="tmdb-key"
                  v-model="draftKey"
                  type="password"
                  placeholder="Paste your TMDB API key"
                  autocomplete="off"
                  spellcheck="false"
                  required
                />

                <button
                  type="submit"
                  :disabled="!draftKey.trim()"
                  class="primary-button"
                >
                  Enter
                  <span>→</span>
                </button>
              </div>

              <p class="key-help">
                Your key stays in this browser tab only.

                <a
                  href="https://www.themoviedb.org/settings/api"
                  target="_blank"
                  rel="noopener"
                >
                  Get a TMDB key ↗
                </a>
              </p>
            </form>

            <!-- SEARCH -->
            <template v-else>
              <form
                class="search-form"
                role="search"
                @submit.prevent="search"
              >
                <label for="movie-query">
                  What are you in the mood for?
                </label>

                <div class="search-field movie-search">
                  <span class="search-icon">⌕</span>

                  <input
                    id="movie-query"
                    v-model="query"
                    type="search"
                    placeholder="Search for a movie..."
                    autocomplete="off"
                  />

                  <button
                    v-if="query"
                    type="button"
                    class="clear-button"
                    @click="clearSearch"
                    aria-label="Clear search"
                  >
                    ×
                  </button>

                  <button
                    :disabled="loading"
                    type="submit"
                    class="primary-button"
                  >
                    Search
                  </button>
                </div>
              </form>

              <div class="quick-actions">
                <button
                  type="button"
                  class="quick-button active"
                  :disabled="loading"
                  @click="popular"
                >
                  ✦ Popular now
                </button>

                <button
                  type="button"
                  class="quick-button"
                  @click="disconnect"
                >
                  Change key
                </button>
              </div>
            </template>

          </div>

          <!-- Decorative visual -->
          <div
            class="hero-decoration"
            aria-hidden="true"
          >
            <div class="orbit orbit-one"></div>
            <div class="orbit orbit-two"></div>

            <div class="film-reel">
              ◉
            </div>

            <span class="floating-star star-one">✦</span>
            <span class="floating-star star-two">✧</span>
            <span class="floating-star star-three">✦</span>
          </div>

        </div>
      </section>

      <!-- MOVIE CATALOG -->
      <section
        id="movies"
        class="catalog"
        aria-labelledby="catalog-title"
        :aria-busy="loading"
      >

        <div class="catalog-header">

          <div>
            <p class="section-label">
              THE COLLECTION
            </p>

            <h2 id="catalog-title">
              {{
                showFavorites
                  ? 'Your favorites'
                  : activeQuery
                    ? `Results for “${activeQuery}”`
                    : 'Popular right now'
              }}
            </h2>
          </div>

          <div class="catalog-actions">

            <button
              class="catalog-button"
              :class="{ selected: !showFavorites }"
              @click="showFavorites = false"
            >
              All movies
            </button>

            <button
              class="catalog-button"
              :class="{ selected: showFavorites }"
              @click="showFavorites = true"
            >
              ♥ Favorites {{ favorites.length }}
            </button>

            <button
              v-if="movies.length"
              class="surprise-button"
              @click="surpriseMe"
            >
              🎲 Surprise me
            </button>

          </div>

        </div>

        <!-- NO API KEY -->
        <div
          v-if="!apiKey"
          class="empty-state"
        >
          <div class="empty-icon">✦</div>

          <h3>
            Your movie universe awaits.
          </h3>

          <p>
            Connect your TMDB API key above to explore
            the collection.
          </p>
        </div>

        <!-- LOADING -->
        <div
          v-else-if="loading"
          class="status-box loading-box"
          role="status"
        >
          <div class="loader"></div>

          <div>
            <strong>Finding movies...</strong>
            <span>Talking to TMDB</span>
          </div>
        </div>

        <!-- ERROR -->
        <div
          v-else-if="error"
          class="status-box error"
          role="alert"
        >
          <div>
            <strong>Something went wrong.</strong>

            <p>
              {{ error }}
            </p>
          </div>

          <button @click="search">
            Try again
          </button>
        </div>

        <!-- FAVORITES EMPTY -->
        <div
          v-else-if="showFavorites && !favorites.length"
          class="empty-state"
        >
          <div class="empty-icon">♥</div>

          <h3>
            No favorites yet.
          </h3>

          <p>
            Tap the heart on any movie to save it here.
          </p>

          <button
            class="primary-button"
            @click="showFavorites = false"
          >
            Browse movies
          </button>
        </div>

        <!-- MOVIES -->
        <template v-else>

          <div
            v-if="(showFavorites ? favorites : movies).length"
            class="movie-grid"
          >
            <MovieCard
              v-for="movie in (showFavorites ? favorites : movies)"
              :key="movie.id"
              :movie="movie"
              :favorite="isFavorite(movie)"
              @toggle-favorite="toggleFavorite"
            />
          </div>

          <div
            v-else
            class="empty-state"
          >
            <div class="empty-icon">?</div>

            <h3>
              No movies found.
            </h3>

            <p>
              Try a different title or return to popular movies.
            </p>

            <button
              class="primary-button"
              @click="popular"
            >
              Browse popular
            </button>
          </div>

          <!-- PAGINATION -->
          <nav
            v-if="!showFavorites && totalPages > 1"
            class="pagination"
            aria-label="Result pages"
          >
            <button
              :disabled="page <= 1"
              @click="loadMovies(page - 1)"
            >
              ← Previous
            </button>

            <span>
              <strong>{{ page }}</strong>
              / {{ totalPages }}
            </span>

            <button
              :disabled="page >= totalPages"
              @click="loadMovies(page + 1)"
            >
              Next →
            </button>
          </nav>

        </template>

      </section>
    </main>

    <!-- FOOTER -->
    <footer
      id="about"
      class="site-footer"
    >
      <div class="footer-main">

        <div>
          <div class="footer-brand">
            MOVIE<span>LAB</span>
          </div>

          <p>
            A cinematic movie discovery interface built with
            Vue and the TMDB API.
          </p>
        </div>

        <div class="footer-meta">
          <span>VUE + TMDB</span>
          <span>WEEK 04</span>
          <span>CLASS PROJECT</span>
        </div>

      </div>

      <div class="footer-bottom">

        <a
          class="tmdb-credit"
          href="https://www.themoviedb.org/"
          target="_blank"
          rel="noopener"
        >
          <img
            :src="tmdbLogo"
            alt="TMDB"
            width="110"
          />
        </a>

        <p>
          This product uses the TMDB API but is not endorsed
          or certified by TMDB.
        </p>

      </div>
    </footer>

  </div>
</template>
