<template>
  <v-card :dark="!!cardColor || $vuetify.theme.dark" :color="cardColor" v-if="value" tile>
    <v-hover v-slot:default="{ hover }">
      <a :href="`#/actor/${value._id}`">
        <v-img
          :cover="fillThumbnail"
          :contain="!fillThumbnail"
          :aspect-ratio="aspectRatio"
          style="cursor: pointer"
          v-ripple
          eager
          :src="thumbnail"
        >
          <v-fade-transition>
            <v-img
              :cover="fillThumbnail"
              :contain="!fillThumbnail"
              eager
              :aspect-ratio="aspectRatio"
              :src="altThumbnail"
              v-if="altThumbnail && hover"
            ></v-img>
          </v-fade-transition>

          <div class="corner-actions">
            <v-btn
              light
              class="elevation-2 mr-1"
              @click.stop.prevent="favorite"
              icon
              style="background: #fafafa"
            >
              <v-icon :color="value.favorite ? 'red' : undefined">{{
                value.favorite ? "mdi-heart" : "mdi-heart-outline"
              }}</v-icon>
            </v-btn>
            <v-btn
              light
              class="elevation-2"
              @click.stop.prevent="bookmark"
              icon
              style="background: #fafafa"
            >
              <v-icon>{{ value.bookmark ? "mdi-bookmark-check" : "mdi-bookmark-outline" }}</v-icon>
            </v-btn>
          </div>
        </v-img>
      </a>
    </v-hover>

    <div class="px-2">
      <v-card-title class="d-flex align-center px-0 pt-1" style="font-size: 1.1rem">
        <Flag class="mr-1" v-if="value.nationality" :width="25" :value="value.nationality.alpha2" />
        <div :title="value.name" class="text-truncate">
          {{ value.name }}
        </div>
        <v-spacer></v-spacer>
        <div class="med--text font-weight-black" v-if="value.bornOn">
          {{ value.age }}
        </div>
      </v-card-title>
      <v-card-subtitle class="pl-0 pb-0"
        >{{ value.numScenes }} {{ value.numScenes == 1 ? "scene" : "scenes" }}</v-card-subtitle
      >
      <Rating @change="rate" class="mb-2" :value="value.rating" />

      <div class="py-1" v-if="value.labels.length && showLabels">
        <label-group :allowRemove="false" :item="value._id" v-model="value.labels" />
      </div>
    </div>
  </v-card>
</template>

<script lang="ts">
import { Component, Vue, Prop } from "vue-property-decorator";
import ApolloClient from "@/apollo";
import gql from "graphql-tag";
import IActor from "@/types/actor";
import { contextModule } from "@/store/context";
import { copy } from "@/util/object";
import { ensureDarkColor } from "@/util/color";

@Component
export default class ActorCard extends Vue {
  @Prop(Object) value!: IActor;
  @Prop({ default: true }) showLabels!: boolean;

  get fillThumbnail() {
    return contextModule.fillActorCards;
  }

  get cardColor() {
    if (this.value.thumbnail && this.value.thumbnail.color)
      return ensureDarkColor(this.value.thumbnail.color);
    return null;
  }

  get aspectRatio() {
    return contextModule.actorAspectRatio;
  }

  rate($event) {
    const rating = $event;

    ApolloClient.mutate({
      mutation: gql`
        mutation($ids: [String!]!, $opts: ActorUpdateOpts!) {
          updateActors(ids: $ids, opts: $opts) {
            rating
          }
        }
      `,
      variables: {
        ids: [this.value._id],
        opts: {
          rating,
        },
      },
    }).then((res) => {
      const actor = copy(this.value);
      actor.rating = res.data.updateActors[0].rating;
      this.$emit("input", actor);
    });
  }

  favorite() {
    ApolloClient.mutate({
      mutation: gql`
        mutation($ids: [String!]!, $opts: ActorUpdateOpts!) {
          updateActors(ids: $ids, opts: $opts) {
            favorite
          }
        }
      `,
      variables: {
        ids: [this.value._id],
        opts: {
          favorite: !this.value.favorite,
        },
      },
    }).then((res) => {
      const actor = copy(this.value);
      actor.favorite = res.data.updateActors[0].favorite;
      this.$emit("input", actor);
    });
  }

  bookmark() {
    ApolloClient.mutate({
      mutation: gql`
        mutation($ids: [String!]!, $opts: ActorUpdateOpts!) {
          updateActors(ids: $ids, opts: $opts) {
            bookmark
          }
        }
      `,
      variables: {
        ids: [this.value._id],
        opts: {
          bookmark: this.value.bookmark ? null : Date.now(),
        },
      },
    }).then((res) => {
      const actor = copy(this.value);
      actor.bookmark = res.data.updateActors[0].bookmark;
      this.$emit("input", actor);
    });
  }

  get thumbnail() {
    if (this.value.thumbnail)
      return `/api/media/image/${this.value.thumbnail._id}?password=${localStorage.getItem(
        "password"
      )}`;
    return "/assets/broken.png";
  }

  get altThumbnail() {
    if (this.value.altThumbnail)
      return `/api/media/image/${
        this.value.altThumbnail._id
      }?password=${localStorage.getItem("password")}`;
    return null;
  }
}
</script>

<style lang="scss" scoped>
.corner-actions {
  position: absolute;
  bottom: 5px;
  left: 5px;
}
</style>