<template>
  <div class="row bg-secondary">
    <img
      v-show="pageOrder > 0"
      @click="subtractFn()"
      class="clickable center-right-image"
      src="@/assets/arrow.png"
      style="width: 120px; z-index: 2"
    />
    <div class="center-left-image">
      <img
        v-show="pageOrder < 3"
        @click="addFn()"
        src="@/assets/arrow.png"
        class="clickable img-flip"
        style="width: 90px"
      />
    </div>
    <div class="container mt-5">
      <div v-show="pageOrder == 0">
        <ButtonDiets
          v-for="dietInFor in diets"
          :key="dietInFor"
          :value="dietInFor"
        />
        <div class="jumbotron jumbotron-fluid my-5">
          <div class="container">
            <h1 class="display-4">{{ store.selectedDiet }}</h1>
            <p class="lead">
              {{ store.dietOpis }}
            </p>
          </div>
        </div>
      </div>
      <div v-show="pageOrder == 1" @click="updateAllergy()">
        <ButtonAllergies
          v-for="allergyInFor in allergies"
          :key="allergyInFor"
          :value="allergyInFor"
        />
        <div class="jumbotron jumbotron-fluid my-5">
          <div class="container">
            <h1 class="display-4">{{ formattedAllergy }}</h1>
            <p class="lead">
              <span v-show="!(store.selectedAllergy[0] === 'Ništa')"
                >Neće se prikazivati hrane koje sadrže:</span
              >
              {{ formattedAllergyDesc }}
            </p>
          </div>
        </div>
      </div>
      <div
        class="container w-75"
        v-show="pageOrder == 2"
        style="margin-bottom: 20%"
      >
        <div class="row" @click="updateOrder()">
          <food v-for="card in cards" :key="card.id" :info="card" />
        </div>
      </div>
      <div v-show="store.totalTime" class="order-menu bg-primary row">
        <p class="col-6">
          Ukupno trajanje vaše narudžbe: <br />{{ store.totalTime }}
          sekundi
        </p>
        <p class="col-6">Vaša narudžba: <br />{{ formattedOrder }}</p>
      </div>

      <div v-show="pageOrder == 3" style="margin-bottom: 50%">
        <stolovi />
      </div>
    </div>
  </div>
</template>

<script>
// @ is an alias to /src
import Food from "@/components/Food.vue";
import Stolovi from "@/components/Stolovi.vue";
import { firebase, db } from "@/firebase.js";
import store from "@/store";
import restrictions from "@/restrictions.js";
import ButtonDiets from "@/components/Button-diets.vue";
import ButtonAllergies from "@/components/Button-allergies.vue";
import { eventBusTables } from "@/main";

export default {
  name: "HomeView",
  data: function () {
    return {
      filterSelect: [],
      cards: [],
      store,
      newImageDescription: "",
      newImageUrl: "",
      pageOrder: 0,
      allergies: null,
      diets: null,
      formattedAllergy: [],
      formattedOrder: [],
      formattedAllergyDesc: [],
    };
  },
  mounted() {
    this.populateAllergies();
    this.populateDiets();
    this.updateAllergy();
  },
  methods: {
    updateOrder() {
      this.formattedOrder = store.currentOrder.join(", ");
    },

    updateAllergy() {
      this.formattedAllergy = store.selectedAllergy.join(", ");
      this.formattedAllergyDesc = store.allergyOpis.join(", ");
    },

    populateDiets() {
      var dietsFilter = restrictions.diets.map((sviNazivi) => {
        return sviNazivi.naziv;
      });
      this.diets = dietsFilter;

      let dijeta = restrictions.diets.find(
        (diet) => diet.naziv === store.selectedDiet
      );
      store.dietOpis = dijeta.opis;
    },

    populateAllergies() {
      var allergiesFilter = restrictions.allergies.map((sviNazivi) => {
        return sviNazivi.naziv;
      });
      this.allergies = allergiesFilter;
    },

    addFn() {
      this.pageOrder++;
      if (this.pageOrder == 2) this.filterFoods();
      if (this.pageOrder == 2) this.getPosts();
      if (this.pageOrder == 3) {
        if (store.totalTime < 1) {
          alert("Odaberite neku hranu!");
          this.pageOrder--;
        } else eventBusTables.$emit("getTables");
      }
    },
    subtractFn() {
      this.pageOrder = this.pageOrder - 1;
    },
    getPosts() {
      db.collection("foods")
        .where("ingredients", "array-contains-any", this.filterSelect)
        .get()
        .then((query) => {
          const includedDocs = query.docs;

          db.collection("foods")
            .orderBy(firebase.firestore.FieldPath.documentId())
            .get()
            .then((queryAll) => {
              const allDocs = queryAll.docs;

              const filteredDocs = allDocs.filter((doc) => {
                return !includedDocs.some(
                  (includedDoc) => includedDoc.id === doc.id
                );
              });

              this.filterSelect = [""];

              this.cards = filteredDocs.map((doc) => {
                const data = doc.data();
                return {
                  name: doc.id,
                  url: data.url,
                  time: data.time,
                  ingredients: data.ingredients.join(", "),
                };
              });
            });
        });
    },
    capitalizeString(string) {
      return string.charAt(0).toUpperCase() + string.slice(1);
    },
    filterFoods() {
      let nazivDijete = restrictions.diets.find(
        (diet) => diet.naziv === this.store.selectedDiet
      );

      this.filterSelect = this.filterSelect.concat(nazivDijete.kategorije);

      let kategorijeDijete = restrictions.categories
        .filter((category) => nazivDijete.kategorije.includes(category.naziv))
        .map((category) => category.sastojci)
        .flat();

      this.filterSelect = this.filterSelect.concat(kategorijeDijete);

      let nazivAlergije;
      let kategorijeAlergije;

      for (let i = store.selectedAllergy.length - 1; i >= 0; i--) {
        nazivAlergije = restrictions.allergies.find(
          (allergy) => allergy.naziv === this.store.selectedAllergy[i]
        );

        this.filterSelect = this.filterSelect.concat(nazivAlergije.kategorije);

        kategorijeAlergije = restrictions.categories
          .filter((category) =>
            nazivAlergije.kategorije.includes(category.naziv)
          )
          .map((category) => category.sastojci)
          .flat();

        this.filterSelect = this.filterSelect.concat(kategorijeAlergije);
      }

      this.filterSelect = this.filterSelect.map((string) =>
        this.capitalizeString(string)
      );

      this.filterSelect = this.filterSelect.filter((value, index, self) => {
        return self.indexOf(value) === index;
      });

      console.log("this.filterSelect: ", this.filterSelect);
    },
  },
  components: {
    Food,
    Stolovi,
    ButtonDiets,
    ButtonAllergies,
  },
};
</script>

<style scoped>
.center-right-image {
  position: fixed;
  top: 50%;
  transform: translate(0%, -50%);
}
.center-left-image {
  position: fixed;
  top: 50%;
  transform: translate(45%, -50%);
}
</style>
