<script lang="ts">
	import { onMount, onDestroy, createEventDispatcher } from 'svelte';
	import SizeAndPositionManager from './SizeAndPositionManager';
	import {
		DIRECTION,
		SCROLL_CHANGE_REASON,
		SCROLL_PROP,
		SCROLL_PROP_LEGACY,
		WRAPPER_MODE,
	} from './constants';

	type Alignment = "auto" | "start" | "center" | "end";
	type ScrollBehaviour = "auto" | "smooth" | "instant";
	type Direction = "horizontal" | "vertical";
	type WrapperMode = "div" | "table";
	type ItemSizeGetter = (index: number) => number;
	type ItemSize = number | number[] | ItemSizeGetter;
	type T = $$Generic;

	export let container: HTMLElement | "self" = null;

	export let height: number | string = '';
	export let width: number | string = '100%';
	
	export let items: T[] = [];
	export let itemCount: number = items.length;
	export let itemSize: ItemSize = 0;
	export let expandItemSize: ItemSize = 0;
	export let estimatedItemSize: number = null;
	export let estimatedExpandItemSize: number = null;
	export let stickyIndices: number[] = null;
	export let getKey: (index: number) => any = null;
	
	export let overflowYMode: "visible" | "hidden" | "auto" | "scroll" = "auto";
	export let overflowXMode: "visible" | "hidden" | "auto" | "scroll" = "auto";
	export let scrollDirection: Direction = 'vertical';
	export let scrollOffset: number = null;
	export let scrollToIndex: number = null;
	export let scrollToAlignment: Alignment = null;
	export let scrollToBehaviour: ScrollBehaviour = 'instant';

	export let overscanCount: number = 15;
	export let preloadScreens: number = 3;
	export let scrollVelocityThreshold: number = 30;
	
	let lastScrollDirection = 0;
	let scrollVelocity = 0;
	let lastScrollTimestamp = 0;
	let rafId = null;
	let isScrolling = false;
	let scrollEndTimeout = null;
	const SCROLL_END_DELAY = 150;
	export let mode: WrapperMode = 'div';

	let expandItems: boolean[] = expandItemSize !== 0 ? new Array(items.length || itemCount).fill(false) : [];

	export function onClick(index) {
		if (expandItemSize === 0) return;
		expandItems[index] = !expandItems[index];
	}

	const dispatchEvent = createEventDispatcher();

	const sizeAndPositionManager = new SizeAndPositionManager({
		itemCount,
		itemSize,
		expandItems,
		expandItemSize,
		estimatedItemSize: getEstimatedItemSize(),
		estimatedExpandItemSize: getEstimatedExpandItemSize(),
	});

	let mounted = false;
	let wrapper = null;
	let scrollWrapper = null;
	let header = null, headerHeight = 0, originalHeight = null;
	let visibleItems: {index: number, style: Record<string, string>}[] = [];

	let state = {
		offset:             scrollOffset || (scrollToIndex != null && items.length && getOffsetForIndex(scrollToIndex)) || 0,
		scrollChangeReason: SCROLL_CHANGE_REASON.REQUESTED,
	};

	let prevState = state;
	let prevProps = {
		scrollToIndex,
		scrollToAlignment,
		scrollOffset,
		itemCount,
		itemSize,
		expandItems: [...expandItems],
		expandItemSize,
		estimatedItemSize,
		estimatedExpandItemSize,
		container,
		items,
	};

	let styleCache = {};
	let wrapperStyle = '';
	let innerStyle = '';

	let containerSize = 0;

	let scrollTimeout = null;
	let lastScrollTime = 0;
	let lastScrollOffset = 0;
	const SCROLL_DEBOUNCE = 16; // ~60fps

	// Track items length separately to avoid reacting to internal item changes
	let prevItemsLength = items.length;

	$: {
		/* listen to updates: */mounted, scrollToIndex, scrollToAlignment, scrollOffset, itemCount, itemSize, expandItems, expandItemSize, estimatedItemSize, estimatedExpandItemSize, container;
		propsUpdated();
	}

	// Ensure items is always an array
	$: items = Array.isArray(items) ? items : [];
	
	// Ensure itemCount is valid
	$: itemCount = Math.max(0, itemCount);

	// Separate reactive statement for items array changes
	$: if (mounted && items && (items.length !== prevItemsLength || items !== prevProps.items)) {
		prevItemsLength = items.length;
		if (itemCount === prevProps.items?.length) {
			itemCount = items.length;
		}
		
		if (expandItemSize !== 0 && expandItems.length !== items.length) {
			expandItems = new Array(items.length).fill(false);
		}
		
		sizeAndPositionManager.updateConfig({
			itemCount: Math.max(0, itemCount), // Ensure positive
			itemSize,
			expandItems,
			expandItemSize,
			estimatedItemSize: getEstimatedItemSize(),
			estimatedExpandItemSize: getEstimatedExpandItemSize(),
		});
		
		// Clear caches and force complete refresh
		styleCache = {};
		wrapperStyle = ''; // Clear wrapper style to force recalculation
		innerStyle = ''; // Clear inner style to force recalculation
		refresh();
		
		// Force update container size and total height
		const totalSize = sizeAndPositionManager.getTotalSize();
		updateContainerSize(totalSize);
	}

	$: {
		/* listen to updates: */ mounted, state;
		stateUpdated();
	}

	$: {
		/* listen to updates: */ mounted, height, width, stickyIndices;
		if (mounted) recomputeSizes(0); // call scroll.reset;
	}

	refresh(); // Initial Load

	onMount(() => {
		header = wrapper.querySelector('[slot="header"]');
		
		if (header) headerHeight = header.offsetHeight;
		
		mounted = true;
		
		containerPropUpdated();

		if (scrollOffset != null) {
			scrollTo(scrollOffset);
		} else if (scrollToIndex != null) {
			scrollTo(getOffsetForIndex(scrollToIndex));
		}
	});

	onDestroy(() => {
		if (mounted) {
			if (scrollTimeout) {
				clearTimeout(scrollTimeout);
			}
			if (scrollEndTimeout) {
				clearTimeout(scrollEndTimeout);
			}
			if (rafId) {
				cancelAnimationFrame(rafId);
			}
			window.removeEventListener('resize', handleScroll);
			if (scrollWrapper === document.body) {
				window.removeEventListener('scroll', handleScroll);
			} else {
				scrollWrapper.removeEventListener('scroll', handleScroll);
			}
		}
	});

	function propsUpdated() {
		if (!mounted) return;
		const scrollPropsHaveChanged =
			      prevProps.scrollToIndex !== scrollToIndex ||
			      prevProps.scrollToAlignment !== scrollToAlignment;
		const itemPropsHaveChanged =
			      prevProps.itemCount !== itemCount ||
			      prevProps.itemSize !== itemSize ||
				  prevProps.expandItems !== expandItems || 
				  prevProps.expandItemSize !== expandItemSize || 
			      prevProps.estimatedItemSize !== estimatedItemSize ||
				  prevProps.estimatedExpandItemSize !== estimatedExpandItemSize ||
				  prevProps.items !== items;
		const containerPropChanged = prevProps.container !== container;

		if (itemPropsHaveChanged) {
			sizeAndPositionManager.updateConfig({
				itemCount,
				itemSize,
				expandItems,
				expandItemSize,
				estimatedItemSize: getEstimatedItemSize(),
				estimatedExpandItemSize: getEstimatedExpandItemSize(),
			});

			if (containerPropChanged) containerPropUpdated();

			recomputeSizes();
		}

		if (prevProps.scrollOffset !== scrollOffset) {
			state = {
				offset:             scrollOffset || 0,
				scrollChangeReason: SCROLL_CHANGE_REASON.REQUESTED,
			};
		} else if (
			typeof scrollToIndex === 'number' &&
			(scrollPropsHaveChanged || itemPropsHaveChanged)
		) {
			state = {
				offset: getOffsetForIndex(
					scrollToIndex,
					scrollToAlignment,
					itemCount,
				),
				scrollChangeReason: SCROLL_CHANGE_REASON.REQUESTED,
			};
		}

		prevProps = {
			scrollToIndex,
			scrollToAlignment,
			scrollOffset,
			itemCount,
			itemSize,
			expandItems: [...expandItems],
			expandItemSize,
			estimatedItemSize,
			estimatedExpandItemSize,
			container,
			items,
		};
	}

	function containerPropUpdated() {
		if (prevProps.container && scrollWrapper) {
			scrollWrapper.classList.remove("virtual-list-container");
			if (originalHeight) {
				scrollWrapper.style.setProperty("height", originalHeight);
				originalHeight = null;
			}
		}
		
		if (container && typeof container === 'object' && 'nodeType' in container) {
			scrollWrapper = container;
		} else if (container === "self") {
			scrollWrapper = wrapper;
		}
		
		if (scrollWrapper) {
			originalHeight = scrollWrapper.style.height;
			scrollWrapper.style.setProperty("height", getVisibleHeight(scrollWrapper) + "px", "important");
			scrollWrapper.classList.add("virtual-list-container");
		} else {
			if ((scrollDirection === DIRECTION.VERTICAL && height) || (scrollDirection === DIRECTION.HORIZONTAL && width)) {
				scrollWrapper = wrapper;
				//scrollWrapper.style.setProperty("overflow", "auto", "important");
			} else {
				scrollWrapper = document.querySelector("body");
			}
		}
		
		window.addEventListener('resize', handleScroll);
		if (scrollWrapper === document.body) {
			window.addEventListener('scroll', handleScroll);
		} else {
			scrollWrapper.addEventListener('scroll', handleScroll);
		}
	}

	function stateUpdated() {
		if (!mounted) return;

		const { offset, scrollChangeReason } = state;

		if (
			prevState.offset !== offset ||
			prevState.scrollChangeReason !== scrollChangeReason
		) {
			refresh();
		}

		if (prevState.offset !== offset && scrollChangeReason === SCROLL_CHANGE_REASON.REQUESTED) {
			scrollTo(offset);
		}

		prevState = state;
	}

	function refresh() {
		const { offset } = state;
		
		const newContainerSize = getVisibleHeight(scrollWrapper === document.body ? wrapper : scrollWrapper);
		if (Math.abs(newContainerSize - containerSize) > 1) {
			containerSize = newContainerSize;
		}
		
		const { start, stop } = sizeAndPositionManager.getVisibleRange(
			containerSize,
			offset,
			mode === WRAPPER_MODE.TABLE ? overscanCount * 2 : overscanCount,
		);
		
		let updatedItems = [];
		const totalSize = sizeAndPositionManager.getTotalSize();
		
		// Always update container size on refresh
		updateContainerSize(totalSize);

		const hasStickyIndices = stickyIndices != null && stickyIndices.length !== 0;
		if (hasStickyIndices) {
			for (let i = 0; i < stickyIndices.length; i++) {
				const index = stickyIndices[i];
				if (index >= 0 && index < items.length) {
					updatedItems.push({
						index,
						style: getStyle(index, true),
					});
				}
			}
		}

		if (start !== undefined && stop !== undefined) {
			for (let index = start; index <= stop; index++) {
				if (index < 0 || index >= items.length) continue;
				if (hasStickyIndices && stickyIndices.includes(index)) continue;

				updatedItems.push({
					index,
					style: getStyle(index, false),
				});
			}

			dispatchEvent('itemsUpdated', {
				start,
				end: stop,
				totalSize,
			});
		}

		visibleItems = updatedItems;
	}

	function scrollTo(value) {
		if (!scrollWrapper || value === undefined || value === null) return;
		
		if (scrollDirection === DIRECTION.VERTICAL && !height) {
			value += wrapper?.offsetTop || 0;
		}

		if ('scroll' in scrollWrapper) {
			scrollWrapper.scroll({
				[SCROLL_PROP[scrollDirection]]: Math.max(0, value),
				behavior: scrollToBehaviour,
			});
		} else {
			scrollWrapper[SCROLL_PROP_LEGACY[scrollDirection]] = Math.max(0, value);
		}
	}

	export function recomputeSizes(startIndex = 0) {
		styleCache = {};
		sizeAndPositionManager.resetItem(startIndex);
		refresh();
	}

	function getOffsetForIndex(index, align = scrollToAlignment, _itemCount = itemCount) {
		if (index < 0 || index >= _itemCount) {
			index = 0;
		}

		const containerSize = getVisibleHeight(scrollWrapper === document.body ? wrapper : scrollWrapper);
		return sizeAndPositionManager.getUpdatedOffsetForIndex(
			align,
			Number(containerSize),
			state.offset || 0,
			index,
		);
	}

	function handleScroll(event) {
		if (event.target !== scrollWrapper && event.target !== document) return;
		
		const currentOffset = getWrapperOffset();
		if (state.offset === currentOffset) return;
		
		const now = Date.now();
		const timeDiff = Math.max(1, now - lastScrollTime);
		
		// Calculate scroll velocity and direction
		const delta = currentOffset - lastScrollOffset;
		const newDirection = Math.sign(delta);
		scrollVelocity = Math.abs(delta) / timeDiff;
		
		// Set scrolling state
		isScrolling = true;
		if (scrollEndTimeout) {
			clearTimeout(scrollEndTimeout);
		}
		
		// Adjust overscan based on scroll velocity
		const velocityMultiplier = Math.min(4, 1 + (scrollVelocity / scrollVelocityThreshold));
		const dynamicOverscan = Math.min(100, Math.max(overscanCount, 
			Math.ceil(overscanCount * velocityMultiplier)
		));
		
		// If direction changed, immediately update to prevent blank areas
		if (newDirection !== lastScrollDirection) {
			lastScrollDirection = newDirection;
			updateScrollState(currentOffset, dynamicOverscan);
			return;
		}
		
		if (timeDiff < SCROLL_DEBOUNCE) {
			if (scrollTimeout) {
				return;
			}
			
			scrollTimeout = setTimeout(() => {
				scrollTimeout = null;
				lastScrollTime = Date.now();
				updateScrollState(currentOffset, dynamicOverscan);
			}, SCROLL_DEBOUNCE - timeDiff);
			
			return;
		}
		
		lastScrollTime = now;
		updateScrollState(currentOffset, dynamicOverscan);
		
		// Set timeout to detect when scrolling ends
		scrollEndTimeout = setTimeout(() => {
			isScrolling = false;
			// Refresh with normal overscan when scrolling ends
			updateScrollState(currentOffset, overscanCount);
		}, SCROLL_END_DELAY);
	}

	function updateScrollState(currentOffset, dynamicOverscan) {
		if (Math.abs(currentOffset - lastScrollOffset) < 1) return;
		
		lastScrollOffset = currentOffset;
		
		if (rafId) {
			cancelAnimationFrame(rafId);
		}
		
		rafId = requestAnimationFrame(() => {
			rafId = null;
			state = {
				offset: currentOffset,
				scrollChangeReason: SCROLL_CHANGE_REASON.OBSERVED,
			};
			
			const extraOverscan = Math.ceil(containerSize * (isScrolling ? preloadScreens : 1));
			const visibleStart = Math.max(0, currentOffset - extraOverscan);
			const visibleSize = containerSize + (extraOverscan * 2);
			
			const { start, stop } = sizeAndPositionManager.getVisibleRange(
				visibleSize,
				visibleStart,
				dynamicOverscan
			);
			
			updateVisibleItems(start, stop);
			
			dispatchEvent('afterScroll', {
				offset: currentOffset,
			});
		});
	}

	function updateVisibleItems(start, stop) {
		if (start === undefined || stop === undefined || !items) return;
		
		// Ensure valid range
		start = Math.max(0, start);
		stop = Math.min(items.length - 1, stop);
		
		let updatedItems = [];
		const hasStickyIndices = stickyIndices != null && stickyIndices.length !== 0;
		
		// Add sticky items first
		if (hasStickyIndices) {
			for (let i = 0; i < stickyIndices.length; i++) {
				const index = stickyIndices[i];
				if (index >= 0 && index < items.length) {
					updatedItems.push({
						index,
						style: getStyle(index, true),
					});
				}
			}
		}
		
		// Add visible and overscan items
		for (let index = start; index <= stop; index++) {
			if (index < 0 || index >= items.length) continue;
			if (hasStickyIndices && stickyIndices.includes(index)) continue;
			
			updatedItems.push({
				index,
				style: getStyle(index, false),
			});
		}
		
		visibleItems = updatedItems;
		
		dispatchEvent('itemsUpdated', {
			start,
			end: stop,
		});
	}

	function getContainerSize() {
		let visibleHeight = 0;
		if (wrapper) {
			const rect = wrapper.getBoundingClientRect();
			visibleHeight = Math.min(rect.bottom, window.innerHeight) - Math.max(rect.top, 0);
		}

		const containerSize = scrollDirection === DIRECTION.VERTICAL ? (height || visibleHeight) : width;
		
		return containerSize;
	}

	function getWrapperOffset() {
		if (!scrollWrapper) return 0;
		
		if (scrollWrapper === document.body) {
			const scrollTop = window.pageYOffset || document.documentElement.scrollTop;
			const wrapperTop = wrapper ? wrapper.getBoundingClientRect().top + scrollTop : 0;
			return scrollTop - wrapperTop - headerHeight;
		}
		
		const scrollTop = scrollDirection === DIRECTION.VERTICAL ? 
			scrollWrapper.scrollTop : 
			scrollWrapper.scrollLeft;
		
		const wrapperTop = getDistanceToParent(wrapper, scrollWrapper);
		return scrollTop - wrapperTop - headerHeight;
	}

	function getEstimatedItemSize() {
		return (
			estimatedItemSize ||
			(typeof itemSize === 'number' && itemSize) ||
			50
		);
	}

	function getEstimatedExpandItemSize() {
		return (
			estimatedExpandItemSize ||
			(typeof expandItemSize === 'number' && expandItemSize) ||
			50
		);
	}

	function getStyle(index, sticky) {
		if (index < 0 || index >= items.length) return null;
		
		const cacheKey = `${index}-${sticky}`;
		if (styleCache[cacheKey]) return styleCache[cacheKey];
		
		const { size, offset, expandSize, expandOffset } = sizeAndPositionManager.getSizeAndPositionForIndex(index);		
		if (size === undefined || offset === undefined) return null;
		
		let style, expandStyle;

		if (mode === WRAPPER_MODE.TABLE) {
			style = `height:${size}px;`;
			expandStyle = `height:${expandSize}px;`;

			if (sticky) {
				style += `position:sticky;z-index:1;top:0;`;
				expandStyle += `position:sticky;z-index:1;top:0;`;
			}
		} else {
			if (scrollDirection === DIRECTION.VERTICAL) {
				style = `left:0;width:100%;height:${size}px;`;
				expandStyle = `left:0;width:100%;height:${expandSize}px;`;

				if (sticky) {
					style += `position:sticky;flex-grow:0;z-index:1;top:0;margin-top:${offset}px;margin-bottom:${-(offset + size)}px;`;
					expandStyle += `position:sticky;flex-grow:0;z-index:1;top:0;margin-top:${expandOffset}px;margin-bottom:${-(expandOffset + expandSize)}px;`;
				} else {
					style += `position:absolute;top:${offset}px;will-change:top;`;
					expandStyle += `position:absolute;top:${expandOffset}px;will-change:top;`;
				}
			} else {
				style = `top:0;height:100%;width:${size}px;`;
				expandStyle = `top:0;height:100%;width:${expandSize}px;`;

				if (sticky) {
					style += `position:sticky;z-index:1;left:0;margin-left:${offset}px;margin-right:${-(offset + size)}px;`;
					expandStyle += `position:sticky;z-index:1;left:0;margin-left:${expandOffset}px;margin-right:${-(expandOffset + expandSize)}px;`;
				} else {
					style += `position:absolute;left:${offset}px;will-change:left;`;
					expandStyle += `position:absolute;left:${expandOffset}px;will-change:left;`;
				}
			}
		}

		return styleCache[cacheKey] = {
			style,
			expandStyle,
		};
	}

	function getVisibleHeight(element) {
		if (!element) return;
		const rect = element.getBoundingClientRect();
		const visibleTop = Math.max(rect.top, 0);
		const visibleBottom = Math.min(rect.bottom, window.innerHeight);
		const visibleHeight = visibleBottom - visibleTop;

		return Math.max(visibleHeight, 0);
	}

	function getDistanceToParent(child, parent) {
		let distance = 0;
		let currentElement = child;
		while (currentElement && currentElement !== parent && currentElement !== document.body) {
			distance += currentElement.offsetTop;
			currentElement = currentElement.offsetParent;
		}
		return distance;
	}

	function updateContainerSize(totalSize) {
		if (scrollDirection === DIRECTION.VERTICAL) {
			const heightUnit = typeof height === 'number' ? 'px' : '';
			const heightValue = height ? (height + heightUnit) : '100%';
			const widthUnit = typeof width === 'number' ? 'px' : '';
			wrapperStyle = `height:${heightValue};width:${width}${widthUnit};position:relative;overflow:hidden;`;
			if (mode === WRAPPER_MODE.TABLE) {
				innerStyle = `width:100%;position:relative;height:${totalSize}px;`;
			} else {
				innerStyle = `flex-direction:column;height:${totalSize}px;width:100%;${height ? 'position:absolute;' : 'position:relative;'}`;
			}
		} else {
			const heightUnit = typeof height === 'number' ? 'px' : '';
			const heightValue = height ? (height + heightUnit) : '100%';
			const widthUnit = typeof width === 'number' ? 'px' : '';
			wrapperStyle = `height:${heightValue};width:${width}${widthUnit};`;
			if (mode === WRAPPER_MODE.TABLE) {
				innerStyle = `width:${totalSize}px;height:100%;`;
			} else {
				innerStyle = `flex-direction:row;width:${totalSize}px;height:100%;${height ? 'position:absolute;' : 'position:relative;'}`;
			}
		}
	}
</script>

<div bind:this={wrapper} class="virtual-list-wrapper overflow-x-{overflowXMode} overflow-y-{overflowYMode}" style={wrapperStyle}>
	<slot name="header" />
	{#if mounted}
		{#if mode === WRAPPER_MODE.DIV}
			<div class="virtual-list-inner" style={innerStyle}>
				{#each visibleItems as item (getKey ? getKey(item.index) : item.index)}
					<div style={item.style.style} class={expandItems[item.index] ? "active" : ""}>
						<slot name="item" item={items[item.index]} index={item.index} style="height: 100%;" />
					</div>
					{#if expandItems[item.index]}
						<div style={item.style.expandStyle}>
							<slot name="expandItem" item={items[item.index]} index={item.index} style="height: 100%; overflow: hidden;" />
						</div>
					{/if}
				{/each}
			</div>
		{:else}
			<table class="virtual-list-inner" style={innerStyle}>
				<thead>
					<slot name="header-row" />
				</thead>
				<tbody>
					<!-- Top spacer with explicit inline style to override the class -->
					<tr style="height:{state.offset}px !important; min-height:{state.offset}px !important;" />
					
					{#each visibleItems as item (getKey ? getKey(item.index) : item.index)}
						<slot 
							name="item" 
							item={items[item.index]} 
							index={item.index}
							style={item.style.style}
							class={expandItems[item.index] ? "active" : ""}
						/>
						{#if expandItems[item.index]}
							<slot 
								name="expandItem" 
								item={items[item.index]} 
								index={item.index}
								style={item.style.expandStyle}
							/>
						{/if}
					{/each}

					<!-- Bottom spacer with explicit inline style -->
					<tr style="height:{Math.max(0, sizeAndPositionManager.getTotalSize() - state.offset - containerSize)}px !important; min-height:{Math.max(0, sizeAndPositionManager.getTotalSize() - state.offset - containerSize)}px !important;" />
				</tbody>
			</table>
		{/if}
	{/if}

	<slot name="footer" />
</div>

<style>
	.virtual-list-wrapper {
		will-change: transform;
		-webkit-overflow-scrolling: touch;
	}

	.virtual-list-inner {
		position: relative;
		width: 100%;
	}

	table.virtual-list-inner {
		border-collapse: collapse;
		table-layout: fixed;
		margin: 0;
		padding: 0;
		position: relative;
	}

	:global(.virtual-list-container) {
		position: relative !important;
		-webkit-overflow-scrolling: touch !important;
		overflow: auto !important;
	}
	:global(.virtual-list-wrapper:not(.virtual-list-container).overflow-x-scroll) {
		overflow-x: scroll !important;
	}
	:global(.virtual-list-wrapper:not(.virtual-list-container).overflow-x-auto) {
		overflow-x: auto !important;
	}
	:global(.virtual-list-wrapper:not(.virtual-list-container).overflow-x-hidden) {
		overflow-x: hidden !important;
	}
	:global(.virtual-list-wrapper:not(.virtual-list-container).overflow-x-visible) {
		overflow-x: visible !important;
	}
	:global(.virtual-list-wrapper:not(.virtual-list-container).overflow-y-scroll) {
		overflow-y: scroll !important;
	}
	:global(.virtual-list-wrapper:not(.virtual-list-container).overflow-y-auto) {
		overflow-y: auto !important;
	}
	:global(.virtual-list-wrapper:not(.virtual-list-container).overflow-y-hidden) {
		overflow-y: hidden !important;
	}
	:global(.virtual-list-wrapper:not(.virtual-list-container).overflow-y-visible) {
		overflow-y: visible !important;
	}

	:global(.virtual-list-spacer) {
		padding: 0 !important;
		border: none !important;
		height: 0 !important;
		line-height: 0 !important;
	}

	table.virtual-list-inner :global(td) {
		box-sizing: border-box;
	}

	table.virtual-list-inner :global(tr) {
		margin: 0;
		padding: 0;
		position: relative;
	}
</style>