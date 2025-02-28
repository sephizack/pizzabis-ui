<template>
  <v-container>
    <v-row
      style="margin: auto; width: 99%; margin-bottom: -20px; margin-top: 10px"
    >
      <v-text-field
        outlined
        label="Recherchez une recette ou des ingredients"
        prepend-inner-icon="search"
        v-model="curFreeSearch"
      ></v-text-field>
    </v-row>
    <v-row class="mt-0 ml-2 text-left" style="margin-bottom: -20px">
      <v-chip
        v-for="category in topLevelFilters"
        :key="category"
        class="ma-2"
        :color="filteredTopLevelCategory == category ? 'green' : 'darkgrey'"
        outlined
        @click="toogleCategory(category)"
      >
        {{ category }}
        <v-icon
          v-if="filteredTopLevelCategory == category"
          color="lightgreen"
          right
          >filter_alt</v-icon
        >
        <v-icon v-else color="grey" right>filter_alt_off</v-icon>
      </v-chip>
    </v-row>
    <v-row
      v-if="subLevelFilters.length > 0"
      class="mt-4 ml-2 text-left"
      style="margin-bottom: -20px"
    >
      <v-tooltip bottom>
        <template v-slot:activator="{ on, attrs }">
          <v-chip
            v-for="category in subLevelFilters"
            :key="category"
            class="ma-2"
            :color="filteredSubLevelCategory == category ? 'green' : 'darkgrey'"
            outlined
            v-bind="attrs"
            v-on="on"
            @click="toogleSubCategory(category)"
          >
            {{ category }}
            <v-icon
              v-if="filteredSubLevelCategory == category"
              color="lightgreen"
              right
              >filter_alt</v-icon
            >
            <v-icon v-else color="grey" right>filter_alt_off</v-icon>
          </v-chip>
        </template>
        <span>Cliquez pour filter</span>
      </v-tooltip>
    </v-row>
    <v-row class="text-center">
      <PizzaCard
        v-for="recipe in filteredRecipes"
        :key="recipe.name"
        :recipe-data="recipe"
        :cart="cart"
        @addToCart="addToCart($event)"
        @removeFromCart="removeRecipeFromCart($event)"
      />
    </v-row>
    <RecipesCart
      ref="recipesCart"
      :cart-data="cart"
      :recipes-data="recipesMap"
      @addRecipe="addToCart($event)"
      @removeRecipe="removeRecipeFromCart($event)"
      @emptyCart="emptyCart"
    />
  </v-container>
</template>

<script>
import PizzaCard from "./PizzaCard";
import RecipesCart from "./RecipesCart";
import recipesRawData from "../assets/recipes.json";

export default {
  name: "RecipesList",
  components: {
    PizzaCard,
    RecipesCart,
  },
  data: () => ({
    filteredTopLevelCategory: "",
    filteredSubLevelCategory: "",
    curFreeSearch: "",
    recipesMap: {},
    recipes: [],
    categories: [],
    cart: {},
    topLevelFilters: [],
    subLevelFilters: [],

  }),
  computed: {
    filteredRecipes: function () {
      let rep = [];
      let search = this.curFreeSearch.toLowerCase();
      for (let recipe of this.recipes) {
        if (
          this.filteredTopLevelCategory != "" &&
          recipe.path.indexOf(this.filteredTopLevelCategory) == -1
        ) {
          continue;
        }
        if (
          this.filteredSubLevelCategory != "" &&
          recipe.path.indexOf(this.filteredSubLevelCategory) == -1
        ) {
          continue;
        }
        if (this.curFreeSearch != "") {
          let aSearchBase = Object.keys(recipe.ingredients).join(", ");
          aSearchBase += ", " + recipe.name;
          if (aSearchBase.toLowerCase().indexOf(search) == -1) {
            continue;
          }
        }
        rep.push(recipe);
      }
      return rep;
    },
  },
  methods: {
    updateRecipeData() {
      fetch('./recipes.json')
        .then(response => response.json())
        .then(data => {
          this.integrateRecipesData(data);
        })
        .catch(error => {
          console.error('Error loading recipes:', error);
          // Fallback to local data if fetch fails
          this.integrateRecipesData(recipesRawData);
        });
  },
    integrateRecipesData: function (data) {
      this.computeRecipeData([], data);
      this.computeTopLevelFilters(data);
      this.computeSubLevelFilters(data);
    },
    computeTopLevelFilters: function (recipesData) {
      let rep = [];
      for (let cat in recipesData) {
        rep.push(cat);
      }
      return rep;
    },
    computeSubLevelFilters: function (recipesData) {
      let rep = [];
      if (this.filteredTopLevelCategory == "") {
        return rep;
      }
      let catData = recipesData[this.filteredTopLevelCategory];
      for (let subcat in catData) {
        if (
          typeof catData[subcat] == "object" &&
          !("recipe" in catData[subcat])
        ) {
          rep.push(subcat);
        }
      }
      return rep;
    },
    toogleCategory: function (category) {
      this.filteredSubLevelCategory = "";
      this.filteredTopLevelCategory =
        this.filteredTopLevelCategory == category ? "" : category;
    },
    toogleSubCategory: function (category) {
      this.filteredSubLevelCategory =
        this.filteredSubLevelCategory == category ? "" : category;
    },
    addNewRecipe: function (curTypePath, recipeName, recipeData) {
      recipeData["name"] = recipeName;
      recipeData["path"] = curTypePath.join(" > ");
      this.recipes.push(recipeData);
      this.recipesMap[recipeName] = recipeData;
    },
    computeRecipeData: function (curTypePath, data) {
      for (let key in data) {
        let value = data[key];
        if (typeof value == "object") {
          if ("recipe" in value) {
            this.addNewRecipe(curTypePath, key, value);
          } else {
            this.categories.push(key);
            this.computeRecipeData(curTypePath.concat(key), value);
          }
        }
      }
      // Sort by ranking
      this.recipes.sort(function (first, second) {
        return second.rating - first.rating;
      });
    },
    saveCart: function () {
      localStorage.setItem("cart", JSON.stringify(this.cart));
    },
    addToCart: function (recipename) {
      if (!this.cart[recipename]) {
        this.$root.$set(this.cart, recipename, 1);
      } else {
        this.$root.$set(this.cart, recipename, this.cart[recipename] + 1);
      }
      this.saveCart();
    },
    openCart: function () {
      this.$refs.recipesCart.open();
    },
    removeRecipeFromCart: function (recipeName) {
      console.log("removing", recipeName);
      if (!this.cart[recipeName]) {
        return;
      } else {
        this.$root.$set(this.cart, recipeName, this.cart[recipeName] - 1);
      }
      if (this.cart[recipeName] == 0) {
        this.$root.$delete(this.cart, recipeName);
      }
      this.saveCart();
    },
    emptyCart: function () {
      for (let recipe in this.cart) {
        this.$root.$delete(this.cart, recipe);
      }
      this.saveCart();
    },
  },
  created: function () {
    this.updateRecipeData()
    try {
      let parsedStorage = JSON.parse(localStorage.getItem("cart"));
      if (parsedStorage) {
        this.cart = parsedStorage;
      }
    } catch (e) {
      console.log("bite");
    }
  },
};
</script>
