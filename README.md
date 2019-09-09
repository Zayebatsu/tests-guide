# Style guide of frontend tests in Shoplo projects


* [Typy testów](#test-types)
  * [Testy jednostkowe](#unit-tests)
  * [Testy komponentów](#components-tests)
  * [Testy integracyjne](#integration-tests)
  * [Testy E2E](#e2e-tests)
* [Struktura plików](#files-structure)
* [Lokalne Vue](#local-vue)
  * [Tłumaczenia](#local-vue-i18n)
* [Renderowanie komponentu](#component-render)
* [Mockowanie](#mocking)
  * [Funkcji](#mocking-functions)
  * [API (Axios)](#mocking-axios)
  * [Stanu aplikacji (Vuex)](#mocking-vuex)
  * [Routera](#mocking-router)
  * [Fixturki](#fixtures)
* [Przykłady częstych testów](#examples-of-tests)
  * [Testowanie routera](#router-tests)
  * [Testowanie Vuex](#vuex-tests)
* [Sprawdzenie wywołania funckji](#spy-on-functions)


* [Flush promises](#flush-promises)
* [Snapshot test](#snapshot-test)

## Typy testów
Prócz standardowych testów manualnych powinniśmy testować nasze aplikacje za pomocą testów automatycznych. W zależności od tego co chcemy przetestować możemy podzielić testy na następujące typy:

### Testy jednostkowe
Testy jednostkowe są na najniższym poziomie abstrakcji i powinny testować pojedyncze maksymalnie wyizolowane części kodu. W naszym przypadku testy jednostkowe są najczęściej wykorzystowane do testowania Vuexa, filtrów oraz pojedynczych funkcji/metod.

### Testy komponentów
W tym przypadku testujemy pojedynczy komponent. W testach powinny znaleźć się testy poprawnego wczytywania komponentu, wszystkich danych wejściowych (propsy), emity, poszczególne metody jeśli to konieczne oraz testy snapshotów.

### Testy integracyjne
Testy integracyjne przeprowadzamy na całych stronach które zawierają inne komponenty, serwisy, stan aplikacji itp. W przypadku tych testów musimny zamockować lokalne Vue (o tym później), stan aplikacji oraz całą komunikacje API.

### Testy E2E

## Struktura plików
Testy jednostkowe, testy komponentów oraz testy integracyjne umieszczamy w katalogu \_\_tests\_\_ w katalogu testowanego pliku. Dzięki temu mamy w jendym miejscu kod który jest testowany oraz kod testu.
Każdy plik powinien zawierać blok descrbe którego tytuł będzie informował nas o komponencie który testujemy. Jeśli testujemy większy komponent i mamy po kilka testów do poszczególnych testów lub potrzebujemy dla kilku testów stworzyć różne mocki możemy grupować testy za pomocą bloków describe.

```js
describe('Products list page', () => {
  // Test body
  describe('Filter products', () => {

    test('When filtering products by availability filter ensure the list is filtered by correct filters', () => {

    });
  });
})
```

Dzięki temu przy debugowaniu testów będziemy dokładnie wiedzieć w którym miejscu test nie działa.

Products list page -> Filter products -> When filtering products by availability filter ensure the list is filtered by correct filters

## Lokalne Vue
Najwięcej pewności dają testy komponentów które są wyrenderowane w najbardziej realistycznym środowisku. Dlatego też przy renderowaniu komponentów do metody `mount`
dodajemy lokalne Vue.
Lokalne Vue to kopia Vue stworzona za pomocą metody `createLocalVue`. Dzięki lokalnemu Vue możemy zainicjalizować pluginy i biblioteki z których korzystamy globalnie w naszej aplikacji. Najczęściej są to pluginy tłumaczeń, walidacji oraz biblioteki UI.

Przykładowy plik z localVue:

```ts
import { createLocalVue } from '@vue/test-utils';
import ShoploKitVue from '@shoplo/kit-vue';
import VeeValidate from 'vee-validate';
import Notifications from 'vue-notification';
import VueI18n from 'vue-i18n';
import i18n from './locale';
import VueRouter from 'vue-router';
import '@/directives/money';
import '@/directives/numeric';
import '@/filters/MoneyFormat';
import '@/filters/DateFormat';

const localVue = createLocalVue();
if (!process || process.env.NODE_ENV !== 'test') {
  localVue.use(VueRouter);
}
localVue.use(VueI18n);
localVue.use(Notifications);
localVue.use(ShoploKitVue, { i18n });
localVue.use(VeeValidate, {
  i18nRootKey: 'lang.validation',
  i18n,
  inject: false,
});

export default localVue;
```

### Tłumaczenia
Aby w testach nasze klucze były przetłumaczenie, należy zainicjaliować plugin tłumaczeń, użyć go w localVue oraz dodać do opcji `mount`.

Plik z tłumaczeniami:
```ts
import Vue from 'vue';
import en from '@/locale/i18n/inpost_en-GB.json';
import VueI18n from 'vue-i18n';
Vue.use(VueI18n);
const i18n = new VueI18n({
  locale: 'en',
  silentTranslationWarn: true,
  messages: {
    en: {
      lang: en,
    },
  },
});

export default i18n;
```

## Renderowanie komponentu
Dobrą praktyką, szczególnie podczas testowania dużych komponentów i testów intergracyjncyh jest stworzenie sobie funkcji renderowania komponentu.

Przykład:

```js
describe('Parcel creation tests', () => {
  let wrapper: any = null;

  const renderComponent = async (mocks: any) => {
    mockAxios.reset();
    mockAxiosStore.reset();
    localVue.use(Vuex);

    const store = new Vuex.Store(storeConfig);
    const params = {
      method: 'shop.checkout.findPointDataByExternalIdAndType',
      external_id: 'WAW07B',
      type: 'PACZKOMAT'
    };
    mockAxios.onGet('https://store.shoplo.com/ajax', params).reply(200, 'elo');
    mockAxios.onGet('settings').reply(200, settings);
    mockAxios.onGet('shop').reply(200, shop);
    mockAxios.onPost('parcels/locker').reply(200, {id: '3213-13123-123-123'});

    await store.dispatch('fetchSettings');
    await store.dispatch('fetchShop');

    wrapper = mount(ParcelCreation, {
      mocks,
      localVue,
      i18n,
      store,
      router,
    });
    await flushPromises();
  };

  afterAll(() => {
    mockAxios.restore();
    mockAxiosStore.restore();
    wrapper = null;
  });

  describe('Test fetch required data on page load', () => {
    beforeEach(async () => {
      // Definiowanie dodatkowych opcji do metody mount
      const mocks = {
        $route: {
          name: 'parcelCreation',
          params: {
            id: orderId,
            order: actionOrders[0],
          },
        },
        $router,
      };
      // Wywołanie funkcji renderowania komponenty
      await renderComponent(mocks);
    });
  });
});
```


## Mockowanie
Pod pojęciem mockowania kryje się przejmowanie kontroli nad daną częścią kodu i tworzenia jej atrapy która zachowa się w zdefiniowany przez nas sposób.

### Mockowanie funkcji
Do mockowania funkcji najczęściej używamy 2 metod które oferuje nam jest: `jest.mock` do mockowania modułów/plików oraz `jest.fn` do mockowania poszczególnych funkcji.

Przykłady:
```js
// Mockowanie funkcji z lokalnego komponentu
wrapper.vm.getProducts = jest.fn(() => {
  // func body
});
// Mockowanie modułu z wewnątrz projektu
jest.mock('@/shared/Backend-validation', () => () => '');

// Mockowanie zewnętrznego modułu z npm
jest.mock('file-saver', () => () => '');
```

### Mockowanie API
Komunikację API możemy zamockować na poziomie funkcji lub serwisu korzystając z `jest.mock` bądź `jest.fn`. Wtedy całkowicie pomijamy zapytania HTTP.
IMO lepszym rozwiązaniem jest mockowanie konkretnych zapytań do API. Do tego możemy użyć biblioteki [axios-mock-adapter](https://github.com/ctimmerm/axios-mock-adapter).
Dzięki niej jesteśmy w stanie przechwycić zapytania HTTP które testowany kod będzie wywoływał i nadpisać ich odpowiedź z serwera.
Jest to o tyle dobre rozwiązanie ponieważ nie trzeba nadpisywać żadnych funkcji i wykluczać zapytań HTTP które aplikacja wywołuje. W prosty sposób można zwrócić odpowiedzi ze statusami np. 200, 404, 403, 500 itp.

Przykład:
```js
import axios from 'axios';
const mockAxios = new MockAdapter(axios);

describe('Parcel details page', () => {
  afterAll(() => {
    mockAxios.restore();
  });

  beforeEach(() => {
    mockAxios.reset();
    mockAxios.onGet('settings').reply(200, settings);
    mockAxios.onGet('shop').reply(200, shop);
    mockAxios.onGet('products', { params: { ids: '4,48' } }).reply(200, products);
    mockAxios.onGet('parcels/label/zpl', {params: { parcelIds: [ parcelId ], isReturn: false }}).reply(200, '');
    mockAxios.onPost('parcels/locker').reply(200, {id: '3213-13123-123-123'});
  });
});
```
Jeśli używamy serwisów do Axiosa ponieważ np. zawsze musimy wysyłać dane autoryzacyjne możemy stworzyć mock tego serwisu axiosa
```js
import MockAdapter from 'axios-mock-adapter';
import { axiosService } from '@/shared/Axios.service';
const mockAxios = new MockAdapter(axiosService);

export { mockAxios };
```

Gdzie `axiosService` to:
```js
import axios from 'axios';
export const axiosService = axios.create({
  baseURL: appInfo.appUrl + '/webapp/',
  headers: {
    'Content-Type': 'application/json',
    'jwt': appInfo.token,
    'app-id': appInfo.shopId,
  },
});
```

### Mockowanie stanu aplikacji
Jeśli chcemy wyrenderować nasz komponent razem z modułem Vuex należy przed zamontowaniem komponentu dodać `localVue.use(Vuex)` oraz stworzyć mock naszego stanu. Mock naszego stanu (store) tworzymy importując nasz realny kod obsługi Vuex.
```js
import localVue from '@/../tests/unit/setup/localVue';
import { actions, getters, mutations } from '@/store/Store';
const storeConfig = {
  actions,
  getters,
  mutations,
};

const renderComponent = async () => {
  localVue.use(Vuex);
  const store = new Vuex.Store(storeConfig);

  // fetchSettings to nazwa konkretnej akcji którą wywołujemy przy renderowaniu komponenty
  store.dispatch('fetchSettings');

  // jeśli akcja w vuex jest asynchroniczna i chcemy poczekać aż się wykona możemy użyć async/await
  await store.dispatch('fetchShop');

  // jeśli funkcja akcji przyjmuje argumenty możemy dodać tam np. fixturki
  store.dispatch('setCreatedParcels', createdParcels);

  const wrapper = mount(ParcelDetails, {
    localVue,
    store,
  });
};
```

### Mockowanie Routera
Są 2 sposoby użycia routera w testach:
- użycie routera w

### Fixturki
Fixturki najlepiej trzymać jako osobne pliki, mogą to być obiekty lub funkcje tworzące obiekty. Dobrą praktyką jest by testy miały osobne fixturki, nawet jeśli część fixturek będzie się powtarzać.

## Sprawdzenie wywołania funkcji

```js
test('Download dispatch order manifest', () => {
  // Przypisanie śledzenia metody komponentu do zmiennej
  const spy = jest.spyOn(wrapper.vm, 'downloadManifest');

  const button = wrapper
    .findAll('[data-cy="download-manifest"]')
    .at(2);

  // Wywołanie clicku na guziku który powinien wywołać testowną metodę
  button.trigger('click');

  expect(button.attributes('disabled')).toBe('disabled');

  // Sprawdzenie czy metoda została wywołana
  expect(spy).toHaveBeenCalled();
});
```

## Przykłady częstych testów

### Testowanie routera


### Testowanie Vuex
