<script setup lang="ts">
interface Props {
  info: string
  link: string
  linkLabel: string
}

const props = withDefaults(defineProps<Props>(), {
  info: "Thomas Thorstensson",
  link: "https://github.com/thorstensson",
  linkLabel: "Repository",
})
</script>

<template>
  <div class="footer">
    {{ props.info }} &nbsp;
    <a :href="props.link" target="_blank" rel="noopener noreferrer">{{ props.linkLabel }}</a>
  </div>
</template>

<style lang="scss" scoped>
a {
  text-decoration: underline;
}
.footer {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  height: 50px;
  padding: 0 32px;
  flex-shrink: 0;
  font-family: $sans-ui;
  font-weight: 400;
  font-variation-settings: "wght" 400;
  font-size: clamp-calc(480px, 1920px, 12px, 14px);
  color: $secondary;
  transition: color 0.2s ease-in-out;
  cursor:pointer;

}

.footer:hover {
  color: $accent1;
}
</style>
