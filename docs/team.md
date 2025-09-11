---
layout: page
---
<script setup>
import {
  VPTeamPage,
  VPTeamPageTitle,
  VPTeamMembers, VPTeamPageSection
} from 'vitepress/theme'

const members = [
  {
    avatar: '/assets/staff/Boreas618.png',
    name: '孙一',
    desc: ' ',
    links: [
      { icon: 'github', link: 'https://github.com/Boreas618' },
    ]
  },
  {
    avatar: '/assets/staff/JingYiJun.png',
    name: '陈可',
    desc: ' ',
    links: [
      { icon: 'github', link: 'https://github.com/JingYiJun' },
    ]
  },
  {
    avatar: '/assets/staff/kooWZ.png',
    name: '孔令宇',
    desc: ' ',
    links: [
      { icon: 'github', link: 'https://github.com/kooWZ' },
    ]
  },
  {
    avatar: '/assets/staff/tangjiewei0336.png',
    name: '唐傑伟',
    desc: '为什么演奏春日影',
    links: [
      { icon: 'github', link: 'https://github.com/tangjiewei0336' },
    ]
  },
  {
    avatar: '/assets/staff/fduTristin.png',
    name: '徐厚泽',
    desc: 'Keep exploring.',
    links: [
      { icon: 'github', link: 'https://github.com/fduTristin' },
    ]
  },
]
</script>

<VPTeamPage>

<VPTeamPageSection>
  <template #title>助教</template>
  <template #members>
    <VPTeamMembers size="small" :members="members" />
  </template>
</VPTeamPageSection>

</VPTeamPage>
