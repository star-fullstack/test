<template>
  <q-page class="q-pa-lg">
    <div class="column flex-center q-mx-auto">
      <q-btn
        dense
        icon="refresh"
        class="q-pa-xs q-mb-md"
        outline
        label="Show Random"
        color="purple"
        @click="getNewQuote"
        aria-label="Menu"
      />
      <div v-if="quote != null">
        <q-card class="my-card" style="width: 400px;">
          <q-card-section>
            <div class="text-h6 q-mb-xs">{{ quote.author }}</div>
            <div class="row no-wrap items-center">
              <q-rating
                size="18px"
                readonly
                v-model="quote.rating"
                :max="5"
                color="primary"
              />
              <span class="text-caption text-grey q-ml-sm"
                >{{ quote.rating }} ({{ quote.numberOfVotes }})</span
              >
            </div>
          </q-card-section>
          <q-separator />
          <q-card-section>
            <q-badge>English</q-badge>
            <div class="shadow-1 q-pa-md">{{ quote.en }}</div>
          </q-card-section>
          <q-card-section v-if="quote.otherLang">
            <q-badge class="text-capitalize">{{ quote.otherLang }}</q-badge>
            <div class="shadow-1 q-pa-md">{{ quote.sr }}</div>
          </q-card-section>
        </q-card>
      </div>
      <div class="q-pa-md row items-center" v-if="quote != null">
        <q-rating
          v-model="myRating"
          size="3.5em"
          class="col"
          color="green-5"
          icon="star_border"
          icon-selected="star"
        />
        <div>
          <q-btn
            dense
            class="q-pa-xs q-ml-md"
            color="primary"
            label="Submit"
            icon="thumb_up"
            @click="submitRating"
            aria-label="Menu"
          />
        </div>
      </div>
      <div v-if="quotes != null">
        <q-card
          v-for="_quote in quotes"
          :key="_quote.id"
          class="my-card"
          style="width: 400px;"
        >
          <q-card-section>
            <div class="text-h6 q-mb-xs">{{ quote.author }}</div>
            <div class="row no-wrap items-center">
              <q-rating
                size="18px"
                readonly
                v-model="quote.rating"
                :max="5"
                color="primary"
              />
              <span class="text-caption text-grey q-ml-sm"
                >{{ quote.rating }} ({{ quote.numberOfVotes }})</span
              >
            </div>
          </q-card-section>
          <q-card-section>
            {{ quote.en }}
          </q-card-section>
        </q-card>
      </div>
      <q-inner-loading :showing="isLoading">
        <q-spinner-gears size="50px" color="primary" />
      </q-inner-loading>
    </div>
  </q-page>
</template>

<script>
import stringSimilarity from "string-similarity";
import LanguageDetect from "languagedetect";
const lngDetector = new LanguageDetect();

export default {
  name: "PageIndex",
  data() {
    return {
      quote: null,
      quotes: null,
      myRating: 1,
      isLoading: true,
    };
  },
  computed: {},
  mounted() {
    this.getNewQuote();
  },
  methods: {
    reset() {
      this.myRating = 1;
    },
    async getNewQuote() {
      try {
        this.isLoading = true;
        const rndQuote = await this.getRandomQuote();
        await this.getQuoteDetail(rndQuote.id);
        this.isLoading = false;
      } catch (error) {
        this.$q.notify(
          "There are some issues when calling API! Please try again!"
        );
      }
    },
    async getRandomQuote() {
      try {
        const { data } = await this.$axios.get(
          "https://programming-quotes-api.herokuapp.com/quotes/random"
        );
        return data;
      } catch (error) {
        this.$q.notify(
          "There are some issues when calling API! Please try again!"
        );
      }
    },
    async getQuoteDetail(id) {
      try {
        const { data } = await this.$axios.get(
          `https://programming-quotes-api.herokuapp.com/quotes/id/${id}`
        );
        this.quote = data;
        if (this.quote.sr.length) {
          this.quote.otherLang = this.getLanguage(this.quote.sr);
        }
        return data;
      } catch (error) {
        this.$q.notify(
          "There are some issues when calling API! Please try again!"
        );
      }
    },
    async submitRating(id) {
      try {
        if (this.myRating < 1.0) {
          this.$q.notify("Rating should be greater than 1.0");
          return;
        }
        this.isLoading = true;

        await this.$axios.post(
          "https://programming-quotes-api.herokuapp.com/quotes/vote",
          {
            quoteId: this.quote.id,
            newVote: this.myRating,
          }
        );
        this.quote = await this.getQuoteDetail(this.quote.id);
        if (this.myRating >= 4.0) {
          await this.getSimiliarQuote();
        } else {
          await this.getNewQuote();
        }
        this.reset();
        this.isLoading = false;
      } catch (error) {
        this.$q.notify(
          "There are some issues when calling API! Please try again!"
        );
      }
    },
    async getSimiliarQuote() {
      try {
        let { data } = await this.$axios.get(
          "https://programming-quotes-api.herokuapp.com/quotes"
        );
        data = data
          .filter((element) => element.id != this.quote.id)
          .map((element) => ({
            ...element,
            similarity: stringSimilarity.compareTwoStrings(
              this.quote.en,
              element.en
            ),
          }));
        data.sort((a, b) => b.similarity - a.similarity);
        this.quote = data[0];
      } catch (error) {
        this.$q.notify(
          "There are some issues when calling API! Please try again!"
        );
      }
    },
    getLanguage(str) {
      const data = lngDetector.detect(str, 1);
      return data[0][0];
    },
  },
};
</script>
