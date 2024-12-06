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

	export let container: string = null;

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

	export let overscanCount: number = 3;

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
	};

	let styleCache = {};
	let wrapperStyle = '';
	let innerStyle = '';

	let containerSize = 0;

	$: {
		/* listen to updates: */ scrollToIndex, scrollToAlignment, scrollOffset, itemCount, itemSize, expandItems, expandItemSize, estimatedItemSize, estimatedExpandItemSize, container;
		propsUpdated();
	}

	$: {
		/* listen to updates: */ state;
		stateUpdated();
	}

	$: {
		/* listen to updates: */ height, width, stickyIndices;
		if (mounted) recomputeSizes(0); // call scroll.reset;
	}

	refresh(); // Initial Load

	onMount(() => {
		mounted = true;

		header = wrapper.querySelector('[slot="header"]');

		if (header) headerHeight = header.offsetHeight;

		containerPropUpdated();

		if (scrollOffset != null) {
			scrollTo(scrollOffset);
		} else if (scrollToIndex != null) {
			scrollTo(getOffsetForIndex(scrollToIndex));
		}
	});

	onDestroy(() => {
		if (mounted) {
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
				  prevProps.estimatedExpandItemSize !== estimatedExpandItemSize;
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
		
		if (container) {
			scrollWrapper = document.querySelector(container);
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
		
		containerSize = getVisibleHeight(scrollWrapper === document.body ? wrapper : scrollWrapper);
		
		const { start, stop } = sizeAndPositionManager.getVisibleRange(
			containerSize,
			offset,
			mode === WRAPPER_MODE.TABLE ? overscanCount * 2 : overscanCount,
		);
		
		let updatedItems = [];
		const totalSize = sizeAndPositionManager.getTotalSize();
		
		const isVisibleScroll = overflowXMode === 'visible' && overflowYMode === 'scroll';
		
		if (scrollDirection === DIRECTION.VERTICAL) {
			wrapperStyle = `
				height:${height ? height + 'px' : '100%'};
				width:${width};
				${isVisibleScroll ? 'position:relative;' : ''}
			`;
			
			if (mode === WRAPPER_MODE.TABLE) {
				innerStyle = `
					position:relative;
					width:${isVisibleScroll ? 'max-content' : '100%'};
					min-width:100%;
				`;
			} else {
				innerStyle = `
					position:relative;
					flex-direction:column;
					height:${totalSize}px;
					${isVisibleScroll ? 'width:max-content;min-width:100%;' : ''}
				`;
			}
		} else {
			wrapperStyle = `height:${height ? height + 'px' : '100%'};width:${width}px;${isVisibleScroll ? 'position:relative;' : ''}`;
			if (mode === WRAPPER_MODE.TABLE) {
				innerStyle = `width:${totalSize}px;${isVisibleScroll ? 'min-width:fit-content;' : ''}`;
			} else {
				innerStyle = `min-height:100%;width:${totalSize}px;${isVisibleScroll ? 'min-width:fit-content;' : ''}`;
			}
		}

		const hasStickyIndices = stickyIndices != null && stickyIndices.length !== 0;
		if (hasStickyIndices) {
			for (let i = 0; i < stickyIndices.length; i++) {
				const index = stickyIndices[i];
				updatedItems.push({
					index,
					style: getStyle(index, true),
				});
			}
		}

		if (start !== undefined && stop !== undefined) {
			for (let index = start; index <= stop; index++) {
				if (hasStickyIndices && stickyIndices.includes(index)) {
					continue;
				}

				updatedItems.push({
					index,
					style: getStyle(index, false),
				});
			}

			dispatchEvent('itemsUpdated', {
				start,
				end: stop,
			});
		}

		visibleItems = updatedItems;
	}

	function scrollTo(value) {
		if (scrollDirection === DIRECTION.VERTICAL && !height) {
			value += wrapper.offsetTop;
		}

		if ('scroll' in scrollWrapper) {
			scrollWrapper.scroll({
				[SCROLL_PROP[scrollDirection]]: value,
				behavior:                       scrollToBehaviour,
			});
		} else {
			scrollWrapper[SCROLL_PROP_LEGACY[scrollDirection]] = value;
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
		const offset = getWrapperOffset();
		if (state.offset === offset || (event.target !== scrollWrapper && event.target !== document)) return;
		
		requestAnimationFrame(() => {
			state = {
				offset,
				scrollChangeReason: SCROLL_CHANGE_REASON.OBSERVED,
			};

			dispatchEvent('afterScroll', {
				offset,
				event,
			});
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
		if ('scroll' in scrollWrapper) {
			if (scrollWrapper === document.body) {
				return document.documentElement.scrollTop - getDistanceToParent(wrapper, scrollWrapper) - headerHeight;
			}
			return scrollWrapper.scrollTop - getDistanceToParent(wrapper, scrollWrapper) - headerHeight;
		} else {
			return scrollWrapper[SCROLL_PROP_LEGACY[scrollDirection]];
		}
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
		if (styleCache[index]) return styleCache[index];
		
		const { size, offset, expandSize, expandOffset } = sizeAndPositionManager.getSizeAndPositionForIndex(index);		
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
					style += `position:absolute;top:${offset}px;`;
					expandStyle += `position:absolute;top:${expandOffset}px;`;
				}
			} else {
				style = `top:0;width:${size}px;`;
				expandStyle = `top:0;width:${expandSize}px;`;

				if (sticky) {
					style += `position:sticky;z-index:1;left:0;margin-left:${offset}px;margin-right:${-(offset + size)}px;`;
					expandStyle += `position:sticky;z-index:1;left:0;margin-left:${expandOffset}px;margin-right:${-(expandOffset + expandSize)}px;`;
				} else {
					style += `position:absolute;height:100%;left:${offset}px;`;
					expandStyle += `position:absolute;height:100%;left:${expandOffset}px;`;
				}
			}
		}

		return styleCache[index] = {
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
</script>

<div 
	bind:this={wrapper} 
	class="virtual-list-wrapper overflow-x-{overflowXMode} overflow-y-{overflowYMode} {overflowXMode === 'visible' && overflowYMode === 'scroll' ? 'virtual-list-wrapper-visible-x' : ''}" 
	style="{wrapperStyle} {overflowXMode === 'visible' && overflowYMode === 'scroll' ? 'position: relative; contain: style layout;' : ''}"
>
	<slot name="header" />

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

	:global(.virtual-list-wrapper-visible-x) {
		contain: style layout !important;
		overflow: visible !important;
		overflow-y: scroll !important;
	}

	:global(.virtual-list-wrapper-visible-x .virtual-list-inner) {
		width: max-content !important;
		min-width: 100% !important;
		position: relative !important;
	}

	:global(.virtual-list-wrapper-visible-x > *) {
		position: relative !important;
	}

	:global(.virtual-list-wrapper:not(.virtual-list-container):not(.virtual-list-wrapper-visible-x).overflow-x-scroll) {
		overflow-x: scroll !important;
	}

	:global(.virtual-list-wrapper:not(.virtual-list-container):not(.virtual-list-wrapper-visible-x).overflow-x-auto) {
		overflow-x: auto !important;
	}

	:global(.virtual-list-wrapper:not(.virtual-list-container):not(.virtual-list-wrapper-visible-x).overflow-x-hidden) {
		overflow-x: hidden !important;
	}

	:global(.virtual-list-wrapper:not(.virtual-list-container):not(.virtual-list-wrapper-visible-x).overflow-x-visible) {
		overflow-x: visible !important;
	}

	:global(.virtual-list-wrapper:not(.virtual-list-container):not(.virtual-list-wrapper-visible-x).overflow-y-scroll) {
		overflow-y: scroll !important;
	}

	:global(.virtual-list-wrapper:not(.virtual-list-container):not(.virtual-list-wrapper-visible-x).overflow-y-auto) {
		overflow-y: auto !important;
	}

	:global(.virtual-list-wrapper:not(.virtual-list-container):not(.virtual-list-wrapper-visible-x).overflow-y-hidden) {
		overflow-y: hidden !important;
	}

	:global(.virtual-list-wrapper:not(.virtual-list-container):not(.virtual-list-wrapper-visible-x).overflow-y-visible) {
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