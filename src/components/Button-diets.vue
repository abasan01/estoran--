<template>
  <div class="row justify-content-center">
    <button
      @click="selectDiet(value)"
      type="button"
      class="btn btn-primary btn-custom m-2"
      :disabled="buttonFlag"
    >
      {{ value }}
    </button>
  </div>
</template>

<script>
import store from "@/store";
import restrictions from "@/restrictions";
import { eventBusDiet } from "@/main";

export default {
  data: function () {
    return {
      buttonFlag: false,
    };
  },
  name: "ButtonDiets",
  props: ["value"],
  created() {
    eventBusDiet.$on("ChangeButton", this.checkFlag);
  },
  destroyed() {
    eventBusDiet.$off("ChangeButton", this.checkFlag);
  },
  mounted() {
    this.checkFlag();
  },
  methods: {
    checkFlag() {
      if (this.value == store.selectedDiet) {
        return (this.buttonFlag = true);
      } else return (this.buttonFlag = false);
    },
    selectDiet(buttonValue) {
      store.selectedDiet = buttonValue;
      let dijeta = restrictions.diets.find(
        (diet) => diet.naziv === buttonValue
      );
      console.log(store.dietOpis);
      store.dietOpis = dijeta.opis;
      console.log(store.dietOpis);
      eventBusDiet.$emit("ChangeButton");
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss"></style>
