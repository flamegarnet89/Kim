
<script lang="ts">
  /**
   * Generic tag-filtered card grid for `/blogs/` and `/tutorials/` indexes.
   * Mirrors `themes/sams-theme/layouts/_default/list.html`'s Arrow.js
   * template. Three states per tag:
   *   - undefined → tag not in filter set; card matches by default.
   *   - true      → "include" filter; card must have the tag.
   *   - false     → "exclude" filter; card must NOT have the tag.
   */
  import ReviewCard from '../ReviewCard.svelte';
  import type { Post } from '../../lib/types';

  export interface TaggedPost {
    kind: 'blog';
    name: string;
    link: string;
    weight: number;
    /** ISO-8601 date string. */
    date: string;
    tags: string[];
    description: string;
    img: string;
    img_larger: string;
  }

  type Props = { posts: TaggedEntry[]; base: string };
  export interface TaggedReview {
    kind: 'review';
    name: string;
    link: string;
    date: string;
    tags: string[];
    description: string;
    reviewPost: Post;
  }
  export type TaggedEntry = TaggedPost | TaggedReview;

  let { posts, base }: Props = $props();
  let tagActivations = $state<Record<string, boolean>>({});
  let searchTerm = $state('');
  let viewMode = $state<'cards' | 'list'>('cards');
  let sortOrder = $state<'recent' | 'oldest'>('recent');
  let selectedYear = $state('');
  let yearDropdownOpen = $state(false);

  const availableYears = $derived(
    [...new Set(posts.map((post) => new Date(post.date).getFullYear()))]
      .filter(Number.isFinite)
      .sort((a, b) => b - a),
  );

  const allTags = $derived(
    Array.from(new Set(posts.flatMap((p) => p.tags))).sort(),
  );

  function clickTag(tag: string) {
    if (tag in tagActivations) {
      if (tagActivations[tag] === true) {
        tagActivations = { ...tagActivations, [tag]: false };
      } else {
        const next = { ...tagActivations };
        delete next[tag];
        tagActivations = next;
      }
    } else {
      tagActivations = { ...tagActivations, [tag]: true };
    }
  }

  function activeClass(tag: string): string {
    if (Object.keys(tagActivations).length === 0) return 'active-nan';
    if (tagActivations[tag] === true) return 'active-true';
    if (tagActivations[tag] === false) return 'active-false';
    return 'active-undefined';
  }

  const visiblePosts = $derived.by(() => {
    const words = searchTerm.toLowerCase().trim().split(/\s+/).filter(Boolean);
    return posts
      .filter((post) => {
        if (
          selectedYear &&
          new Date(post.date).getFullYear() !== Number(selectedYear)
        ) {
          return false;
        }

        for (const [tag, active] of Object.entries(tagActivations)) {
          if (active === true && !post.tags.includes(tag)) return false;
          if (active === false && post.tags.includes(tag)) return false;
        }

        const postDate = new Date(post.date);
        const readableDate = Number.isNaN(postDate.getTime())
          ? post.date
          : postDate.toLocaleDateString('en', {
              year: 'numeric',
              month: 'long',
              day: 'numeric',
            });
        const searchableText = [
          post.name,
          post.description,
          ...post.tags,
          ...(post.kind === 'review' ? post.reviewPost.tags : []),
          post.date,
          readableDate,
        ].join(' ').toLowerCase();
        return words.every((word) => searchableText.includes(word));
      })
      .sort((a, b) =>
        sortOrder === 'recent'
          ? b.date.localeCompare(a.date)
          : a.date.localeCompare(b.date),
      );
  });

  const visiblePostsByYear = $derived.by(() => {
    const groups = new Map<string, TaggedEntry[]>();

    for (const post of visiblePosts) {
      const date = new Date(post.date);
      const year = Number.isNaN(date.getTime())
        ? 'Unknown year'
        : String(date.getFullYear());
      const group = groups.get(year) ?? [];
      group.push(post);
      groups.set(year, group);
    }

    return [...groups].map(([year, yearPosts]) => ({ year, posts: yearPosts }));
  });

  function prefixBase(link: string): string {
    if (link.startsWith(base)) return link;
    return base + link;
  }

  function formatDate(date: string): string {
    return new Intl.DateTimeFormat('en', {
      year: 'numeric',
      month: 'long',
      day: 'numeric',
    }).format(new Date(date));
  }
</script>

<div class="mx-auto mt-6 flex max-w-7xl flex-wrap items-center justify-center gap-3 px-4">
 <div role="group" aria-label="Post layout" class="inline-flex overflow-hidden rounded-md">
    <button
      type="button"
      aria-pressed={viewMode === 'cards'}
     class={`px-4 py-2 ${
        viewMode === 'cards'
          ? 'bg-green-500 text-green-50'
          : 'bg-green-700 text-green-400 hover:bg-green-500 hover:text-green-50'
      }`}
      onclick={() => (viewMode = 'cards')}
    >
      Cards
    </button>
    <button
      type="button"
      aria-pressed={viewMode === 'list'}
      class={`px-4 py-2 ${
        viewMode === 'list'
          ? 'bg-green-500 text-green-50'
          : 'bg-green-700 text-green-400 hover:bg-green-500 hover:text-green-50'
      }`}
      onclick={() => (viewMode = 'list')}
    >
      List
    </button>
  </div>
  <div role="group" aria-label="Sort posts" class="inline-flex overflow-hidden rounded-md">
    <button
      type="button"
      aria-pressed={sortOrder === 'recent'}
      class={`px-4 py-2 ${
        sortOrder === 'recent'
          ? 'bg-green-500 text-green-50'
          : 'bg-green-700 text-green-400 hover:bg-green-500 hover:text-green-50'
      }`}
      onclick={() => (sortOrder = 'recent')}
    >
      Recent
    </button>
    <button
      type="button"
      aria-pressed={sortOrder === 'oldest'}
      class={`px-4 py-2 ${
        sortOrder === 'oldest'
          ? 'bg-green-500 text-green-50'
          : 'bg-green-700 text-green-400 hover:bg-green-500 hover:text-green-50'
      }`}
      onclick={() => (sortOrder = 'oldest')}
    >
      Oldest
    </button>
  </div>

  <div class="year-filter-group">
    <button
      type="button"
      class="year-filter-all {selectedYear === '' ? 'is-active' : ''}"
      aria-pressed={selectedYear === ''}
      onclick={() => {
        selectedYear = '';
        yearDropdownOpen = false;
      }}
    >
      All
    </button>
    <div class="year-dropdown">
      <button
        type="button"
        class="year-dropdown-trigger {selectedYear ? 'is-active' : ''}"
        aria-haspopup="listbox"
        aria-expanded={yearDropdownOpen}
        onclick={() => (yearDropdownOpen = !yearDropdownOpen)}
      >
        {selectedYear || 'Year'}
        <svg
          class="h-4 w-4"
          xmlns="http://www.w3.org/2000/svg"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
          aria-hidden="true"
        >
          <path d="m6 9 6 6 6-6" />
        </svg>
      </button>
      {#if yearDropdownOpen}
        <div class="year-dropdown-options" role="listbox" aria-label="Filter posts by year">
          {#each availableYears as year (year)}
            <button
              type="button"
              role="option"
              aria-selected={selectedYear === String(year)}
              onclick={() => {
                selectedYear = String(year);
                yearDropdownOpen = false;
              }}
            >
              {year}
            </button>
          {/each}
        </div>
      {/if}
    </div>
  </div>

  <label class="sr-only" for="post-search">Search posts</label>
  <input
    id="post-search"
    type="search"
    bind:value={searchTerm}
    placeholder="Search posts..."
    class="form-input w-full max-w-sm border-green-600 bg-green-800 text-green-100 placeholder:text-green-400 focus:border-green-400 focus:ring-green-400"
  />
</div>

<div id="tags" class="tag-list max-w-7xl mx-auto mt-6">
  {#each allTags as tag (tag)}
    <button
      type="button"
      class="tag tag-{tag} {activeClass(tag)}"
      onclick={() => clickTag(tag)}
    >
      <span>{tag}</span>
    </button>
  {/each}
</div>

<div id="all-card-wrapper">
  <div class="container mx-auto mt-20 max-w-4xl space-y-8 px-4" aria-label="All posts">
      {#each visiblePostsByYear as group (group.year)}
        <section aria-labelledby="year-{group.year}">
          <h2 id="year-{group.year}" class="mb-3 text-xl font-semibold text-green-100">
            {group.year}
          </h2>
          <ul class="space-y-3">
            {#each group.posts as post (post.link)}
              <li>
                <a
                  class="text-green-300 underline decoration-green-600 underline-offset-4 hover:text-green-100 focus-visible:outline-2 focus-visible:outline-offset-4 focus-visible:outline-green-400"
                  href={prefixBase(post.link)}
                >
                  {#if post.kind === 'review'}Book Review: {/if}{post.name}
                </a>
              </li>
            {/each}
          </ul>
        </section>
      {/each}
    </div>
  {:else}
    <div
      id="card-wrapper"
      class="container mx-auto mt-20 grid gap-12 grid-cols-vertical-cards"
    >
      {#each visiblePosts as post (post.link)}
        {#if post.kind === 'review'}
          <ReviewCard
            post={{ ...post.reviewPost, tags: ['book-review'] }}
            layout="all-posts"
            displayTagLabels={{ 'book-review': 'book review' }}
          />
        {:else}
          <div class="fancy_card horizontal mx-auto w-full" data-tagged-card>
            <div class="card_translator">
              <a class="card_rotator small_rot card_layer block" href={prefixBase(post.link)} aria-label={post.name}>
                <div class="card_layer">
                  <article class="flex flex-col h-full bg-green-700">
                    <header class="mb-4">
                      <figure class="relative h-0 pb-9/16 overflow-hidden">
                        <picture>
                          <source
                            srcset="{post.img} 352w, {post.img_larger} 704w"
                            type="image/webp"
                          />
                          <img
                            width="352"
                            height="198"
                            class="absolute inset-0 w-full h-full object-cover"
                            src={post.img}
                            alt={post.name}
                          />
                        </picture>
                      </figure>
                    </header>
                    <div class="px-4 pb-4 flex flex-col flex-grow min-h-[13.25rem]">
                      <div class="mb-3">
                        <ul class="flex flex-wrap font-medium -m-1 justify-center">
                          {#each post.tags as tag (tag)}
                            <li class="m-1 inline-flex text-center py-1 px-3 rounded-full tag-{tag}">
                              {tag}
                            </li>
                          {/each}
                        </ul>
                      </div>
                      <h4 class="mb-2">{post.name}</h4>
                      <p class="text-sm text-green-300 mb-2">{formatDate(post.date)}</p>
                      <p class="text-lg text-green-300 flex-grow">{post.description}</p>
                    </div>
                  </article>
                </div>
                <div class="card_layer card_effect card_overlay_C"></div>
                <div class="card_layer card_effect card_glare"></div>
              </a>
            </div>
          </div>
        {/if}
      {/each}
    </div>
  {/if}
</div>
