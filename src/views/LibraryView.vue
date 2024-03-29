<script setup lang="ts">
import { onMounted, ref, watch } from "vue";
import { useRouter, useRoute } from "vue-router";
import { useProductStore, useCategoryStore, useTopicStore } from "@/store";
import ProductCard from "@/components/ProductCard.vue";
import StarRating from "@/components/StarRating.vue";
import type { Topic } from "@/types/topic";
import type { Product } from "@/types/product";
import type { Category } from "@/types/category";

// Dependencies
const router = useRouter();
const route = useRoute();
const productStore = useProductStore();
const categoryStore = useCategoryStore();
const topicStore = useTopicStore();

// Data Properties
const filteredProducts = ref<Product[]>([]);
const selectedCategories = ref<Category[]>([]);
const selectedTopics = ref<Topic[]>([]);
const searchString = ref<string>("");
const cardsAreLoading = ref<boolean>(false);
const sortOptions = ["A-Z", "Z-A", "Most Popular"];
const selectedSort = ref<string>("A-Z");

// Watchers
watch(selectedCategories, () => {
  appendCategoryQueryString();
  filterProducts();
});
watch(selectedTopics, () => {
  appendCategoryQueryString();
  filterProducts();
});
watch(selectedSort, () => {
  filterProducts();
});

// Methods
function filterProducts() {
  filteredProducts.value = JSON.parse(
    JSON.stringify([...productStore.allActiveProducts])
  ); // reset to starting dataset
  // check for category matches (inclusive)
  if (selectedCategories.value.length > 0) {
    filteredProducts.value = productStore.allActiveProducts.filter((prod) =>
      selectedCategories.value.some((sc: any) => sc.id == prod.category.id)
    );
  }
  // check for topic matches (exclusive)
  if (selectedTopics.value.length > 0) {
    const topicIdArray = selectedTopics.value.map((topic: any) => topic.name);
    filteredProducts.value = filteredProducts.value.filter((prod: any) =>
      topicIdArray.every((name: any) =>
        prod.topics.some((topic: any) => topic.name === name)
      )
    );
  }
  if (searchString.value !== "") {
    filteredProducts.value = filteredProducts.value.filter(
      (prod: any) =>
        prod.name.toLowerCase().includes(searchString.value.toLowerCase()) ||
        prod.description
          .toLowerCase()
          .includes(searchString.value.toLowerCase())
    );
  }
  if (selectedSort.value === "Z-A") {
    filteredProducts.value.reverse();
  } else if (selectedSort.value === "Most Popular") {
    filteredProducts.value = filteredProducts.value.sort((a: any, b: any) => {
      if (a.rating === b.rating) {
        return a.name > b.name ? 1 : -1;
      }
      return b.rating > a.rating ? 1 : -1;
    });
  }
}

function appendCategoryQueryString() {
  router.replace({
    name: "library",
    query: {
      ...route.query,
      categories: selectedCategories.value
        .map((category) => category.name)
        .join(","),
    },
  });
}

function removeCategory(index: number) {
  selectedCategories.value.splice(index, 1);
  appendCategoryQueryString();
  filterProducts();
}

function appendTopicQueryString() {
  router.replace({
    name: "library",
    query: {
      ...route.query,
      topics: selectedTopics.value.map((topic: any) => topic.name).join(","),
    },
  });
}

// function goToReviews(e: Event, productId: string) {
//   e.stopPropagation();
//   e.preventDefault();
//   router.push(`/product/${productId}/reviews`);
// }

function removeTopic(index: number) {
  selectedTopics.value.splice(index, 1);
  appendTopicQueryString();
  filterProducts();
}

function searchByString() {
  router.replace({
    name: "library",
    query: {
      ...route.query,
      search: searchString.value,
    },
  });
  filterProducts();
}

function filterByQueryString(
  queryCategories: any,
  queryTopics: any,
  querySearch: any
) {
  if (queryCategories) {
    queryCategories = queryCategories
      .split(",")
      .map((queryCategory: any) => queryCategory.toLowerCase());
    selectedCategories.value = categoryStore.allCategories.filter((cat: any) =>
      queryCategories.includes(cat.name.toLowerCase())
    );
  }
  if (queryTopics) {
    queryTopics = queryTopics
      .split(",")
      .map((queryTopic: any) => queryTopic.toLowerCase());
    selectedTopics.value = topicStore.allTopics.filter((topic: any) =>
      queryTopics.includes(topic.name.toLowerCase())
    );
  }
  if (querySearch) {
    searchString.value = querySearch.toLowerCase();
  }
  filterProducts();
  cardsAreLoading.value = false;
}

// Lifecycle Hooks
onMounted(async () => {
  window.scrollTo(0, 0);
  // read query strings and handle filters as needed
  const queryCategories = route.query.categories || route.query.Categories;
  const queryTopics = route.query.topics || route.query.Topics;
  const querySearch = route.query.search || route.query.Search;
  const queryStringPresent = queryCategories || queryTopics || querySearch;
  cardsAreLoading.value = true;
  categoryStore.fetchCategories();
  topicStore.fetchTopics();

  // show products immediately if they exist in store
  if (productStore.allActiveProducts.length > 0) {
    filteredProducts.value = productStore.allActiveProducts.filter(
      (product: any) => product.isActive == 1
    );
    if (!queryStringPresent) {
      filterByQueryString(queryCategories, queryTopics, querySearch);
    }
  }

  // if products were not in store, render the cards after store is populated
  productStore.getAllProducts();

  if (!queryStringPresent) {
    filterProducts();
  } else {
    filterByQueryString(queryCategories, queryTopics, querySearch);
  }

  cardsAreLoading.value = false;
});
</script>

<template>
  <div class="library">
    <h1 class="heading">Library</h1>

    <div class="chipBar">
      <i class="fas fa-tag"></i>
      <div class="chip-container">
        <span
          v-if="selectedCategories.length + selectedTopics.length < 1"
          class="noTagsText"
        >
          No tags selected.
        </span>
        <v-chip
          v-for="(category, index) in selectedCategories"
          :key="`category${category.id}`"
          closable
          @click:close="removeCategory(index)"
          class="ml-2"
        >
          {{ category.name }}
        </v-chip>
        <v-chip
          v-for="(topic, index) in selectedTopics"
          :key="`topic${topic.name}`"
          closable
          @click:close="removeTopic(index)"
          class="ml-2"
        >
          {{ topic.name }}
        </v-chip>
      </div>
    </div>

    <div class="filterBar">
      <div class="tagFilter-container">
        <!-- Category -->
        <div class="form-group">
          <div class="multiselect-wrapper --library">
            <v-select
              v-model="selectedCategories"
              :items="categoryStore.allCategories"
              multiple
              item-title="name"
              item-value="name"
              label="Categories"
              return-object
            >
              <template v-slot:selection="{ index }">
                <span class="multiselect__single" v-if="index < 1">
                  {{ selectedCategories.length }} selected
                </span>
              </template>
            </v-select>
          </div>
        </div>

        <!-- Topics -->
        <div class="form-group">
          <div class="multiselect-wrapper --library">
            <v-select
              v-model="selectedTopics"
              :items="topicStore.allTopics"
              multiple
              item-title="name"
              item-value="name"
              label="Topics"
              return-object
            >
              <template v-slot:selection="{ index }">
                <span class="multiselect__single" v-if="index < 1">
                  {{ selectedTopics.length }} selected
                </span>
              </template>
            </v-select>
          </div>
        </div>
      </div>

      <!-- Search -->
      <div class="search">
        <i class="fas fa-search"></i>
        <input
          type="search"
          placeholder="Search"
          v-model="searchString"
          @input="searchByString"
        />
      </div>

      <!-- Sort -->
      <div class="multiselect-wrapper sortControl">
        <div class="form-group">
          <v-select
            v-model="selectedSort"
            :items="sortOptions"
            label="Sort"
          ></v-select>
        </div>
      </div>
    </div>

    <!-- Product Cards -->
    <div>
      <transition-group tag="div" name="list" class="cardRow">
        <div
          v-for="product in filteredProducts"
          :key="product.id"
          class="cardContainer"
        >
          <router-link
            :to="{ path: `/product/${product.id}`, query: { ...route.query } }"
          >
            <product-card :details="product">
              <slot>
                <span class="categoryLabel">{{ product.category.name }}</span>
                <div class="bottomRow">
                  <star-rating :rating="product.rating" />
                </div>
              </slot>
            </product-card>
          </router-link>
        </div>
        <div class="emptyRowFiller" key="filler1"></div>
        <div class="emptyRowFiller" key="filler2"></div>
        <div class="emptyRowFiller" key="filler3"></div>
        <div class="emptyRowFiller" key="filler4"></div>
      </transition-group>
    </div>

    <!-- <loading-dots v-if="cardsAreLoading" loading-text="Loading products" /> -->
    <div
      v-if="!cardsAreLoading && filteredProducts.length < 1"
      class="noResults"
    >
      There are no products that match your search.
    </div>
  </div>
</template>

<style lang="scss" scoped>
@import "../assets/scss/mixins";

.library {
  padding: var(--standard-view);
  @include breakpoint(desktop) {
    padding: 4rem 4rem 10rem;
  }
  @include breakpoint(tablet-port) {
    padding: 4rem 2rem 10rem;
  }
}
.heading {
  font-weight: 800;
}

.chipBar {
  display: flex;
  align-items: center;
  margin: 2rem 0;
  min-height: 4rem;
  .chip-container {
    display: flex;
    flex-wrap: wrap;
  }
  .fa-tag {
    color: var(--gray);
    margin-right: 0.5rem;
  }
  .noTagsText {
    color: var(--gray);
    font-size: 12px;
    font-weight: 600;
    margin-left: 1rem;
    line-height: 15px;
  }
}

.filterBar {
  display: flex;
  align-items: flex-end;
  margin-top: 1rem;
  height: 3.7rem;
  @include breakpoint(tablet-port) {
    flex-wrap: wrap;
    height: unset;
  }

  .multiselect-wrapper {
    width: 17rem;
    margin-right: 2rem;
  }
  .tagFilter-container {
    display: flex;
    @include breakpoint(mobile) {
      flex-wrap: wrap;
    }
  }
  .search {
    position: relative;
    margin-left: 1rem;
    @include breakpoint(laptop) {
      position: absolute;
      right: 4rem;
      top: 15.25rem;
      margin: 0;
    }
    @include breakpoint(tablet-port) {
      right: 2rem;
    }
    input {
      width: 23rem;
      height: 3.6rem;
      padding: 0.5rem 0.5rem 0.25rem 2.8rem;
      border-radius: 5rem;
      border: 1px solid var(--gray);
      @include breakpoint(desktop) {
        width: 22rem;
      }
      @include breakpoint(tablet-port) {
        width: 17rem;
      }
      &:focus,
      &:active {
        outline: none;
      }
    }
    .fa-search {
      position: absolute;
      top: 1rem;
      left: 0.75rem;
      color: var(--gray);
    }
  }
  .sortControl {
    margin-left: auto;
    @include breakpoint(tablet-port) {
      margin-left: 0;
    }
  }
}

.cardContainer {
  width: var(--prodCardContainer-width);
  margin: 0 auto;
}

.cardRow {
  display: flex;
  flex-wrap: wrap;
  margin: 5rem 0 0 -2rem;
  padding: 0 4px;

  .categoryLabel {
    color: var(--gray);
    margin-top: 0.25rem;
    font-size: 1.4rem;
  }

  .bottomRow {
    margin-top: 0.5rem;
    display: flex;
    align-items: center;
  }
}
.emptyRowFiller {
  width: var(--prodCardContainer-width);
  height: 0;
  margin: 0 auto;
}
</style>
