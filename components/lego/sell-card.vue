<template>
  <nuxt-link
    :to="{ name: 'order-id', params: { id: order.id } }"
    class="sell-card text-center cursor-pointer"
    :style="{ background: bg }"
  >
    <on-sale-tag v-if="order.onSale && !onlyToken" :time="order.timeleft" />
    <!-- <owned-tag v-if="sellOrderType" /> -->
    <order-type-tag v-if="sellOrderType" :type="sellOrderType" />

    <div class="img-wrapper d-flex ps-t-12 justify-content-center">
      <video v-if="isVideoFormat" autoplay muted loop width="100%">
        <source
          :src="order.token.img_url"
          type="video/webm"
          @error="handleNotVideo"
        />
        <source
          :src="order.token.img_url"
          type="video/ogg"
          @error="handleNotVideo"
        />
        <source
          :src="order.token.img_url"
          type="video/mp4"
          @error="handleNotVideo"
        />
      </video>
      <ImageWithLoader
        v-else
        :src="order.token.image_url"
        class="asset-img align-self-center"
        :alt="order.token.name"
        @load="onImageLoad"
        :fallBackSrc="category.img_url"
      />
    </div>
    <div
      class="gradient"
      :style="{
        background:
          'linear-gradient( 360deg,' + bg + '0%, rgba(236, 235, 223, 0) 100%)',
      }"
    />
    <div v-if="category" class="category-pill d-flex mx-auto ms-t-20 ms-b-16">
      <img
        :src="category.img_url"
        class="icon ms-2 ms-l-4 ms-r-4 align-self-center"
      />
      <div class="font-caps font-medium caps align-self-center ps-r-6">
        {{ category.name }}
      </div>
    </div>
    <h3
      class="w-100 title font-body-small font-medium ms-b-8 ps-x-12"
      :class="{ 'ms-b-16': onlyToken }"
      :title="order.token.name"
    >
      {{ order.token.name }} {{ isErc1155 ? `( ${order.quantity} )` : '' }}
    </h3>
    <div v-if="erc20Token && !onlyToken" class="price font-body-small ms-b-20">
      {{ order.price }} {{ erc20Token.symbol }} &nbsp; ({{ priceInUSD }})
    </div>
  </nuxt-link>
</template>

<script>
import Vue from 'vue'
import Component from 'nuxt-class-component'
import { mapGetters, mapState } from 'vuex'

import rgbToHsl from '~/helpers/color-algorithm'
import { formatUSDValue } from '~/helpers'
import { getColorFromImage } from '~/utils'
import OnSaleTag from '~/components/lego/token/on-sale-tag'
import OwnedTag from '~/components/lego/token/owned-tag'
import OrderTypeTag from '~/components/lego/token/order-type-tag'
import ImageWithLoader from "~/components/common/image";

@Component({
  props: {
    order: {
      type: Object,
      required: true,
    },
    onlyToken: {
      type: Boolean,
      required: false,
      default: false,
    },
    sell: {
      type: Function,
      required: false,
      default: () => {},
    },
    moveToMatic: {
      type: Function,
      required: false,
      default: () => {},
    },
  },
  components: { OnSaleTag, OwnedTag, OrderTypeTag,ImageWithLoader },
  computed: {
    ...mapGetters('category', ['categories', 'categoryById']),
    ...mapGetters('token', ['erc20Tokens']),
    ...mapState('auth', {
      user: (state) => state.user,
    }),
    ...mapState('network', {
      networks: (state) => state.networks,
    }),
  },
  middleware: [],
  mixins: [],
})
export default class SellCard extends Vue {
  bg = '#f3f4f7'
  isVideoFormat = true
  // Initial
  mounted() {

  }

  onImageLoad() {
    try {
      const img = this.$el.querySelector('.asset-img')
      // img.crossOrigin = "Anonymous";

      const rgbColor = getColorFromImage(img)
      if (rgbColor) {
        const hsl = rgbToHsl({
          r: rgbColor[0],
          g: rgbColor[1],
          b: rgbColor[2],
        })
        this.bg = `hsl(${hsl.h},${hsl.s}%,${hsl.l}%)`
      } else {
        this.bg = '#f3f4f7'
      }
    } catch (error) {
      this.$logger.error(error)
      this.bg = '#f3f4f7'
    }
  }

  get erc20Token() {
    return this.erc20Tokens.find(
      (token) => token.id === this.order.erc20TokenId,
    )
  }

  get category() {
    const categoryId = this.order.categories.id
    return this.categoryById(categoryId)
  }

  get isErc1155() {
    return this.order.token_type === 'ERC1155'
  }

  get isErc721() {
    return this.order.token_type === 'ERC721'
  }

  get sellOrderType() {
    return this.order.type
  }

  get priceInUSD() {
    return this.order.usd_price
      ? formatUSDValue(parseFloat(this.order.usd_price))
      : '$0'
  }



  handleNotVideo() {
    this.isVideoFormat = false
  }
}
</script>

<style lang="scss" scoped="true">
@import '~assets/css/theme/_theme';

a {
  color: inherit;
  text-decoration: inherit;
}

.sell-card {
  width: 240px;
  min-height: 352px;
  margin: 0.625rem;
  position: relative;

  background: light-color('700');
  border-radius: $default-card-box-border-radius;
  .img-wrapper {
    width: 100%;
    max-width: 240px;
    height: 100%;
    max-height: 240px;
    overflow: hidden;
    .asset-img {
      max-height: 100%;
      max-width: 100%;
    }
  }
  .gradient {
    width: 100%;
    height: 100px;
    margin-top: -92px;
    position: absolute;
    background: linear-gradient(
      360deg,
      light-color('500') 0%,
      rgba(236, 235, 223, 0) 100%
    );
  }

  .category-pill {
    width: fit-content;
    border-radius: 19px;
    background: light-color('700');
    .icon {
      width: 20px;
      height: 20px;
      border-radius: 50%;
    }
  }
  .title {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .price {
    color: dark-color('400');
  }
}
.sell-card:hover {
  box-shadow: $default-card-box-shadow;
}
.actions {
  height: 45px;
  border-top: 1px solid rgba(dark-color('700'), 0.1);
  &.matic-chain {
    .btn-transparent {
      color: primary-color('600');
      border-radius: 0px;
    }

    .btn-red {
      color: red-color('600');
    }

    .btn:nth-child(2) {
      border-left: 1px solid rgba(dark-color('700'), 0.1);
    }
  }
}
@media (max-width: 330px) {
  .sell-card {
    width: 100%;
    height: auto;
  }
}
</style>
