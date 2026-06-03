<template>
  <div class="page">
    <Header />
    <main class="main">
      <div class="layout-left">
        <breadcrumb
          :info="{
            category_id: id,
            category_locale_name: categoryInfo?.seo_category?.name
          }"
          isCategory
        ></breadcrumb>
        <common-page-label
          :title="`${capitalizeFirstLetter(categoryInfo?.seo_category?.name)} Articles`"
        />
        <div id="relatedsearches1"></div>
        <section>
          <InfiniteLoadList
            api-endpoint="/api/article/get_seo_category_page"
            :initial-page="2"
            :page-size="10"
            :showMore="false"
            :query="{
              seo_category_id: id
            }"
            :initial-items="categoryInfo?.list"
            class="news-box-4"
          >
            <template #default="{ items }">
              <news-item-4
                v-for="(item, index) in items"
                :key="index"
                :index="index"
                :item="item"
              />
            </template>
          </InfiniteLoadList>
        </section>
      </div>
      <div class="layout-right">
        <right-side-box :rec-news="trendingNews?.list" :trending-news="recNews?.list" />
      </div>
    </main>
    <FooterSeo />
  </div>
</template>

<script>
import { capitalizeFirstLetter } from "~/utils/utils";

export default {
  async asyncData({ $axios, params, env }) {
    const path = params.category;
    const lastDashIndex = path.lastIndexOf("-");
    const id = path.substring(lastDashIndex + 1, path.length);

    try {
      const [recNewsResponse, trendingNewsResponse, categoryInfoResponse] = await Promise.all([
        $axios.$get("/api/article/menu", {
          params: { site_id: env.SITE_ID, mod_id: "rec" }
        }),
        $axios.$get("/api/article/get_all_articles", {
          params: { site_id: env.SITE_ID, size: 4, page: 1 }
        }),
        $axios.$get("/api/article/get_seo_category_page", {
          params: { site_id: env.SITE_ID, seo_category_id: id, size: 10, page: 1 }
        })
      ]);
      return {
        recNews: recNewsResponse,
        trendingNews: trendingNewsResponse,
        categoryInfo: categoryInfoResponse,
        id
      };
    } catch (error) {
      console.error("Error fetching data:", error);
      return { recNews: null, trendingNews: null, categoryInfo: null, id };
    }
  },
  head() {
    const category = this.categoryInfo?.seo_category;
    const categoryName = category?.name || "";
    const seoTitle = category?.seo_title || `${categoryName} Articles | Hacksforhome`;
    const seoDesc =
      category?.seo_desc ||
      `Browse the latest ${categoryName} articles on Hacksforhome. Stay informed with comprehensive global news coverage.`;

    const itemListElements =
      this.categoryInfo?.list?.map((item, index) => ({
        "@type": "ListItem",
        position: index + 1,
        url: `https://www.hacksforhome.com/${item.path_v2}/`
      })) || [];

    return {
      title: seoTitle,
      meta: [
        { hid: "description", name: "description", content: seoDesc },
        { hid: "og:title", property: "og:title", content: seoTitle },
        { hid: "og:description", property: "og:description", content: seoDesc },
        {
          hid: "og:url",
          property: "og:url",
          content: `https://www.hacksforhome.com/category/${this.id}/`
        },
        { hid: "og:type", property: "og:type", content: "website" }
      ],
      script: [
        {
          type: "application/ld+json",
          json: {
            "@context": "https://schema.org",
            "@type": "BreadcrumbList",
            itemListElement: [
              {
                "@type": "ListItem",
                position: 1,
                item: { "@id": "https://www.hacksforhome.com/", name: "Home" }
              },
              {
                "@type": "ListItem",
                position: 2,
                item: {
                  "@id": `https://www.hacksforhome.com/category/${this.id}/`,
                  name: categoryName
                }
              }
            ]
          }
        },
        {
          type: "application/ld+json",
          json: {
            "@context": "https://schema.org",
            "@type": "ItemList",
            itemListElement: itemListElements
          }
        }
      ]
    };
  },
  data() {
    return {
      channelId: ""
    };
  },
  mounted() {
    const searchParams = new URLSearchParams(window.location.search);
    if (searchParams.has("channel")) {
      this.channelId = searchParams.get("channel");
    } else {
      this.channelId = this.categoryInfo?.seo_category?.channel || "";
    }
    this.$nextTick(() => {
      this.addAdSenseScript();
    });
  },
  methods: {
    capitalizeFirstLetter,
    addAdSenseScript() {
      const searchParams = new URLSearchParams(window.location.search);
      let terms = searchParams.has("terms") ? searchParams.get("terms") : "";
      terms = terms.replace(/[，]/g, ",");
      let headline = searchParams.has("headline") ? searchParams.get("headline") : "";
      if (headline === "{title}" || headline === "{{ad_title}}") {
        headline = "";
      }

      const paramKeys = [];
      for (const param of searchParams) {
        paramKeys.push(param[0]);
      }
      const ignoredPageParams = paramKeys.join(",");

      const hiSource = window.getParam && window.getParam("hi_source");
      const hiPc = window.getParam && window.getParam("hi_pc");
      const resultsPageBaseUrl = window.getResultsPageUrl && window.getResultsPageUrl({
        channel: this.channelId,
        from: "detail",
        hi_source: hiSource,
        hi_pc: hiPc
      });
      const adSenseConfig = {
        channel: this.channelId,
        pubId: "partner-pub-6612490456597819",
        styleId: "7767580164",
        adsafe: "low",
        ignoredPageParams,
        relatedSearchTargeting: "content",
        resultsPageBaseUrl,
        resultsPageQueryParam: "query",
        terms: terms,
        referrerAdCreative: headline || terms,
        ivt: false,
        adtest: "off"
      };

      // eslint-disable-next-line no-undef
      _googCsa("relatedsearch", adSenseConfig, {
        container: "relatedsearches1",
        relatedSearches: 10,
        adLoadedCallback: function (loaded, response, isExperimentVariant, callbackOptions) {
          if (response) {
            window.trackEventToPixel && window.trackEventToPixel("D_C_AC");
            window.pushEventParamsToGtm && window.pushEventParamsToGtm("C_AC");
            window.setCookie && window.setCookie("query_ad", 1);
            try {
              let numberOfKeys = 0;
              let concatenatedKeys = "miss";
              if (callbackOptions.termPositions) {
                const keys = Object.keys(callbackOptions.termPositions);
                numberOfKeys = keys.length;
                concatenatedKeys = keys.join(",");
              }
              const element = document.getElementById("master-1");
              const height = parseFloat(element.style.height);
              const result = Math.round(height / 105);
              // eslint-disable-next-line no-undef
              dataLayer.push({
                event: "C_AC_IN",
                queryNum: 10,
                num: result,
                key1: numberOfKeys,
                key2: concatenatedKeys
              });
            } catch (e) {
              console.log(e);
            }
          }
        }
      });
    }
  }
};
</script>

<style lang="scss" scoped>
::v-deep .news-box-4 {
  .load-list-content {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 10px 28px;
    font-family: "Noto Sans JP", "Lucida Grande", "Hiragino Kaku Gothic ProN", Meiryo, sans-serif;
  }
}
@media screen and (max-width: 750px) {
  ::v-deep .news-box-4 {
    padding-bottom: vw(32);
    .load-list-content {
      grid-template-columns: repeat(1, 1fr);
      gap: vw(52);
    }
  }
}
</style>
