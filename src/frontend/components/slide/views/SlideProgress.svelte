<script lang="ts">
    import { allOutputs, groups, outputs, showsCache } from "../../../stores"
    import { translateText } from "../../../utils/language"
    import { getActiveOutputs } from "../../helpers/output"
    import { getGroupName, getLayoutRef } from "../../helpers/show"
    import type { LayoutRef, Show, ShowGroups, Slide } from "../../../../types/Show"

    export let tracker: any
    export let outputId = ""
    export let item: any = {}
    export let autoSize = 0

    let type: "number" | "bar" | "group" = "number"
    $: type = tracker.type || "number"
    let accent: string | undefined
    $: accent = tracker.accent

    interface LayoutGroupInfo {
        name: string
        oneLetterName: string
        index: number
        child: number
        hide?: boolean
    }

    let currentShow: Show | null = null
    let currentShowSlides: Record<string, Slide | undefined> = {}
    let currentGroups: ShowGroups = {}
    let currentLayoutRef: LayoutRef[] = []
    let currentOutput: any = {}
    let currentSlideOut: any = null
    let currentShowId = ""
    let currentShowSlide = -1
    let slidesLength = 0
    interface ProgressCacheEntry {
        layoutGroups: LayoutGroupInfo[]
        slidesLength: number
        ref: LayoutRef[]
        groupsSignature: string
    }

    let layoutGroups: LayoutGroupInfo[] = []
    let progressEntry: ProgressCacheEntry = { layoutGroups: [], slidesLength: 0, ref: [], groupsSignature: "" }

    const PROGRESS_CACHE = new Map<string, ProgressCacheEntry>()
    const LAYOUT_CACHE = new Map<string, { seed: string; ref: LayoutRef[] }>()

    $: if (!outputId) outputId = getActiveOutputs()[0]
    $: currentOutput = $outputs[outputId] || $allOutputs[outputId] || {}
    $: currentSlideOut = currentOutput?.out?.slide || null
    $: currentShowId = currentSlideOut?.id || ""
    $: currentShowSlide = currentSlideOut?.index ?? -1
    $: currentShow = currentShowId ? $showsCache[currentShowId] : null
    $: currentLayoutRef = getCachedLayoutRef(currentShowId, currentShow)
    $: currentShowSlides = currentShow?.slides || {}
    $: currentGroups = $groups
    $: progressEntry = getProgressEntry(currentShowId, currentShow, currentLayoutRef, currentShowSlides, currentGroups)
    $: slidesLength = progressEntry.slidesLength
    $: layoutGroups = progressEntry.layoutGroups

    let progressElem: HTMLElement | undefined
    $: column = (progressElem?.offsetWidth || 0) < (progressElem?.offsetHeight || 0)

    function getCacheKey(showId: string) {
        return showId || "__none__"
    }

    function getLayoutSeed(show: Show | null) {
        if (!show) return "0"

        const layoutId = show.settings?.activeLayout || ""
        const layoutSlides = show.layouts?.[layoutId]?.slides || []
        if (!layoutSlides.length) return layoutId

        let seed = `${layoutId}:`
        layoutSlides.forEach((layoutSlide) => {
            const slide = show.slides?.[layoutSlide.id]
            const children = !slide || !Array.isArray(slide.children) ? "" : slide.children.join(",")
            const disabled = layoutSlide?.disabled ? 1 : 0
            const groupValue = slide?.group ?? ""
            const globalGroupValue = slide?.globalGroup ?? ""
            seed += `${layoutSlide.id}:${disabled}:${children}:${groupValue}:${globalGroupValue}|`
        })

        return seed
    }

    function getCachedLayoutRef(showId: string, show: Show | null): LayoutRef[] {
        if (!showId || !show) return []

        const seed = getLayoutSeed(show)
        const cached = LAYOUT_CACHE.get(showId)
        if (cached && cached.seed === seed) return cached.ref

        const ref = getLayoutRef(showId)
        LAYOUT_CACHE.set(showId, { seed, ref })
        return ref
    }

    function getGroupsSignature(groupsStore: ShowGroups | undefined) {
        if (!groupsStore) return "0"
        return Object.entries(groupsStore)
            .map(([id, value]) => `${id}:${value?.name || ""}:${value?.default ? 1 : 0}`)
            .join("|")
    }

    function getProgressEntry(showId: string, show: Show | null, layoutRef: LayoutRef[], slides: Record<string, Slide | undefined>, groupsStore: ShowGroups): ProgressCacheEntry {
        if (!showId || !show || !layoutRef) return { layoutGroups: [], slidesLength: 0, ref: [], groupsSignature: "" }

        const cacheKey = getCacheKey(showId)
        const groupsSignature = getGroupsSignature(groupsStore)
        const cached = PROGRESS_CACHE.get(cacheKey)
        if (cached && cached.ref === layoutRef && cached.groupsSignature === groupsSignature) return cached

        const layoutGroups: LayoutGroupInfo[] = layoutRef.map((refItem) => {
            const parentInfo = refItem.parent || { id: refItem.id, layoutIndex: refItem.layoutIndex }
            const slide = slides[parentInfo.id]
            if (!slide) return { name: "—", oneLetterName: "—", index: parentInfo.layoutIndex, child: 0 }

            if (refItem.data?.disabled || slide.group?.startsWith("~")) {
                return { name: "—", oneLetterName: "—", index: parentInfo.layoutIndex, child: 0, hide: true }
            }

            let groupName: string = slide.group || "—"
            if (slide.globalGroup && groupsStore?.[slide.globalGroup]) {
                const globalGroup = groupsStore[slide.globalGroup]
                groupName = globalGroup.default ? translateText(`groups.${globalGroup.name}`) : globalGroup.name
            }

            if (typeof groupName !== "string") groupName = ""

            const fullName = (getGroupName({ show, showId }, parentInfo.id, groupName, parentInfo.layoutIndex) || "—").replace(/ *\([^)]*\) */g, "")
            const oneLetterSource = groupName ? groupName[0]?.toUpperCase() || "" : ""
            const oneLetterName = (getGroupName({ show, showId }, parentInfo.id, oneLetterSource, parentInfo.layoutIndex) || "—").replace(/ *\([^)]*\) */g, "").replace(" ", "")

            let childIndex = 0
            if (refItem.type === "child") {
                const parentEntry = layoutRef[parentInfo.layoutIndex]
                const parentChildren = Array.isArray(parentEntry?.children) ? parentEntry.children : []
                const childPos = parentChildren.indexOf(refItem.id)
                childIndex = childPos >= 0 ? childPos + 1 : 0
            }

            return {
                name: fullName || "—",
                oneLetterName: oneLetterName || "—",
                index: parentInfo.layoutIndex,
                child: childIndex
            }
        })

        const entry: ProgressCacheEntry = { layoutGroups, slidesLength: layoutRef.length, ref: layoutRef, groupsSignature }
        PROGRESS_CACHE.set(cacheKey, entry)
        return entry
    }
</script>

<div class="progress" bind:this={progressElem} class:barBG={type === "bar"} style={accent ? "--accent: " + accent : ""}>
    {#if type === "number"}
        <div class="align autoFontSize" style="{autoSize ? 'font-size: ' + autoSize + 'px;' : ''}{item?.alignX ? '' : (item?.align || 'justify-content: center;').replaceAll('text-align', 'justify-content')}">
            <span style="color: var(--accent);">{currentShowSlide + 1}</span>/{slidesLength}
        </div>
    {:else if type === "bar"}
        <!-- progress bar -->
        <div class="bar" style="width: {slidesLength ? ((currentShowSlide + 1) / slidesLength) * 100 : 0}%;"></div>
    {:else if type === "group"}
        <!-- group sequence -->
        <div class="align groups autoFontSize" class:column style="{autoSize ? 'font-size: ' + autoSize + 'px;' : ''}{item?.alignX ? '' : (item?.align || 'justify-content: center;').replaceAll('text-align', 'justify-content')}">
            {#each layoutGroups as group}
                {#if !group.child && !group.hide}
                    {@const activeGroup = layoutGroups.find((a, i) => a.index === group.index && i === currentShowSlide)}
                    {@const nextSlide = layoutGroups.find((a, i) => a.index === group.index && i === currentShowSlide + 1)}
                    {@const displayName = tracker.oneLetter ? group.oneLetterName : group.name}

                    <div class="group" class:active={group.index === layoutGroups.find((_, i) => i === currentShowSlide)?.index}>
                        {displayName}{#if tracker.childProgress && (activeGroup?.child || nextSlide?.child)}<span style="opacity: 0.8;font-size: 0.7em;">.{(activeGroup?.child ?? -1) + 1}</span>{/if}
                    </div>
                {/if}
            {/each}
        </div>
    {/if}
</div>

<style>
    .progress {
        width: 100%;
        height: 100%;

        display: flex;
        align-items: center;
        justify-content: center;

        --accent: var(--secondary);
    }

    .progress.barBG {
        justify-content: flex-start;
    }
    .bar {
        height: 100%;
        background-color: var(--accent);
        transition: width 0.5s;
    }

    .align {
        width: 100%;
        height: 100%;
        display: flex;
        align-items: center;
        justify-content: center;
        /* stage align */
        justify-content: var(--text-align);
        outline: none !important;
    }

    .groups {
        display: flex;
        gap: 25px;
        flex-wrap: wrap;
    }
    .groups.column {
        flex-direction: column;
        text-align: start;
        width: 100%;
    }
    .group {
        transition: color 0.2s;
    }
    .group.active {
        color: var(--accent);
    }
</style>
