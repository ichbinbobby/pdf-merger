<script setup>
useHead({
  meta: [
    { name: 'viewport', content: 'width=device-width, initial-scale=1' }
  ],
  link: [
    { rel: 'icon', href: '/favicon.ico' }
  ],
  htmlAttrs: {
    lang: 'en'
  }
})

const title = 'PDF & Image Merger'
const description = 'Merge PDFs and images into a single PDF — entirely in your browser. No uploads, no server, fully private.'
const siteUrl = 'https://pdf.ichbinbobby.de'

useSeoMeta({
  title,
  description,
  author: 'ichbinbobby',
  ogTitle: title,
  ogDescription: description,
  ogUrl: siteUrl,
  ogType: 'website',
  ogSiteName: 'PDF Merger by ichbinbobby',
  ogImage: `${siteUrl}/og-image.png`,
  ogImageWidth: '1200',
  ogImageHeight: '630',
  ogImageAlt: 'PDF & Image Merger — merge files in your browser',
  twitterCard: 'summary_large_image',
  twitterTitle: title,
  twitterDescription: description,
  twitterImage: `${siteUrl}/og-image.png`,
  twitterCreator: '@ichbinbobby'
})

useHead({
  link: [
    { rel: 'canonical', href: siteUrl }
  ]
})
</script>

<template>
  <UApp>
    <UHeader>
      <template #left>
        <NuxtLink
          to="/"
          class="flex items-center gap-2 font-semibold text-gray-900 dark:text-white"
        >
          <UIcon
            name="i-lucide-file-stack"
            class="w-5 h-5 text-primary-500"
          />
          PDF Merger
        </NuxtLink>
      </template>

      <template #right>
        <UColorModeButton />
      </template>
    </UHeader>

    <UMain>
      <NuxtPage />
    </UMain>

    <UFooter>
      <template #left>
        <p class="text-sm text-muted">
          All processing happens in your browser — your files are never uploaded.
        </p>
      </template>
    </UFooter>
  </UApp>
</template>
