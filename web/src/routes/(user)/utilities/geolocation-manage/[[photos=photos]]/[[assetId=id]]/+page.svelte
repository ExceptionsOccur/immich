<script lang="ts">
  import UserPageLayout from '$lib/components/layouts/user-page-layout.svelte';
  import Maplibregl from '$lib/components/shared-components/map/map.svelte';
  import { getMapMarkers, type MapMarkerResponseDto } from '@immich/sdk';
  import GeolocationManageSelect from '$lib/components/utilities-page/geolocation-manage/geolocation-manage-select.svelte';
  import { type ComboBoxOption } from '$lib/components/shared-components/combobox.svelte';
  import {type LngLatBounds} from'maplibre-gl';
  import { t } from 'svelte-i18n';
  import { writable } from 'svelte/store';
  import type { PageData } from './$types';
  import { onMount } from 'svelte';
  import { mapSettings } from '$lib/stores/preferences.store';
  import { DateTime, Duration } from 'luxon';

  interface Props {
    data: PageData;
  }

  let { data = $bindable() }: Props = $props();
  let mapBounds: LngLatBounds;
  let mapMarkers: MapMarkerResponseDto[] = [];
  const mapMarkersStore = writable<MapMarkerResponseDto[]>([]);
  let abortController: AbortController;

  function handleZoom(map: maplibregl.Map) {
    mapBounds = map.getBounds();
  }

  let options = $state([
    {id: 'all', value: 'all', label: '显示所有'},
    {id: 'none', value: 'none', label: '地址为空'}
  ] as ComboBoxOption[])

  let selectedOption = $state(
    {id: 'all', value: 'all', label: '显示所有'}
  )

  function handleSelectedLocation(location: string | undefined){
    let filter: MapMarkerResponseDto[] = [];
    if (location === 'all'){
      mapMarkersStore.set(mapMarkers);
      return;
    }
    else if (location === 'none'){
      filter = mapMarkers.filter(marker => marker.state === '')
    }
    else{
      if(location && location.split(',').length === 1)
        filter = mapMarkers.filter(marker => `${marker.country}` === location)
      else if(location && location.split(',').length === 2)
        filter = mapMarkers.filter(marker => `${marker.country}, ${marker.state}` === location)
      else
        filter = mapMarkers.filter(marker => `${marker.country}, ${marker.state}, ${marker.city}` === location)
    }
    mapMarkersStore.set(filter);
  }

  function getFileCreatedDates() {
    const { relativeDate, dateAfter, dateBefore } = $mapSettings;

    if (relativeDate) {
      const duration = Duration.fromISO(relativeDate);
      return {
        fileCreatedAfter: duration.isValid ? DateTime.now().minus(duration).toISO() : undefined,
      };
    }

    try {
      return {
        fileCreatedAfter: dateAfter ? new Date(dateAfter).toISOString() : undefined,
        fileCreatedBefore: dateBefore ? new Date(dateBefore).toISOString() : undefined,
      };
    } catch {
      $mapSettings.dateAfter = '';
      $mapSettings.dateBefore = '';
      return {};
    }
  }

  async function loadMapMarkers() {
    if (abortController) {
      abortController.abort();
    }
    abortController = new AbortController();

    const { includeArchived, onlyFavorites, withPartners, withSharedAlbums } = $mapSettings;
    const { fileCreatedAfter, fileCreatedBefore } = getFileCreatedDates();

    return await getMapMarkers(
      {
        isArchived: includeArchived && undefined,
        isFavorite: onlyFavorites || undefined,
        fileCreatedAfter: fileCreatedAfter || undefined,
        fileCreatedBefore,
        withPartners: withPartners || undefined,
        withSharedAlbums: withSharedAlbums || undefined,
      },
      {
        signal: abortController.signal,
      },
    );
  }

  onMount(async () => {
    if (!mapMarkers || mapMarkers.length === 0) {
      mapMarkers = await loadMapMarkers();
      let _counties: Map<string, string> = new Map<string, string>();
      let _states: Map<string, string> = new Map<string, string>();
      let _cities: Map<string, string> = new Map<string, string>();
      loadMapMarkers().then((markers) => {
        mapMarkersStore.set(markers);
        mapMarkers = markers;
        markers.map(marker => {
          if(marker.country && !_counties.has(marker.country))
            _counties.set(marker.country, marker.country);
          if (marker.country && marker.state && !_states.has(`${marker.country}, ${marker.state}`))
            _states.set(`${marker.country}, ${marker.state}`, `${marker.country}, ${marker.state}`);
          if (marker.country && marker.state && marker.city && !_cities.has(`${marker.country}, ${marker.state}`))
            _cities.set(`${marker.country}, ${marker.state}, ${marker.city}`, `${marker.country}, ${marker.state}, ${marker.city}`);
        })
        let counties = _counties.keys().map(country => { 
          return {
            id: country,
            value: country,
            label: country
          }
        })
        let states = _states.keys().map(state => { 
          return {
            id: state,
            value: state,
            label: state
          }
        })
        let cities = _cities.keys().map(city => { 
          return {
            id: city,
            value: city,
            label: city
          }
        })
        // let _markers = markers.map(marker => {
        //   if (marker.country && !_counties.has(marker.country)){
        //     _states.set(marker.country, marker.country);
        //     return {
        //       id: marker.country,
        //       value: marker.country,
        //       label: marker.country
        //     }
        //   }
        //   if (marker.country && marker.state && !_states.has(`${marker.country}, ${marker.state}`)){
        //     _states.set(`${marker.country}, ${marker.state}`, `${marker.country}, ${marker.state}`);
        //     return {
        //       id: `${marker.country}, ${marker.state}`,
        //       value: `${marker.country}, ${marker.state}`,
        //       label: `${marker.country}, ${marker.state}`
        //     }
        //   }
        //   return {
        //     id: marker.id,
        //     value: `${marker.country}, ${marker.state}, ${marker.city}`,
        //     label: `${marker.country}, ${marker.state}, ${marker.city}`
        //   }
        // })
        let _options = [...counties, ...states, ...cities]
        options.push(..._options);
      });
    }
  });

</script>

<UserPageLayout title={data.meta.title} scrollbar={true}>
  <div class="relative h-full w-full">
    <div class="absolute border border-gray-300 dark:border-immich-dark-gray pt-1 pb-6 dark:text-white h-full w-1/4 left-0">
        <GeolocationManageSelect
          label='GPS位置'
          comboboxPlaceholder='根据地址筛选'
          {selectedOption}
          {options}
          onSelect={(combobox) => handleSelectedLocation(combobox?.value)}
        />
    </div>
    <div class="absolute h-full w-3/4 right-0">
      <Maplibregl onZoom={handleZoom} hash mapMarkers={$mapMarkersStore} />
    </div>
  </div>
</UserPageLayout>
