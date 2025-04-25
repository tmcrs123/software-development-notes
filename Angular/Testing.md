# HTTP Testing Controller

Need to  `provideHttpClient` and `provideHttpClientTesting`. Flow is a bit different. You don't mock the `HttpClient` itself, but rather it's mocked by Angular behind the scenes. Then you need to call `flush` to return the result of the HTTP call.

```javascript
    beforeEach(async () => {

        await TestBed.configureTestingModule({
            providers: [
                provideHttpClient(),
                provideHttpClientTesting(),
                provideMockStore({ initialState })
            ]

        }).compileComponents()

        httpTesting = TestBed.inject(HttpTestingController);
        service = TestBed.inject(MoviesService);
        store = TestBed.inject(Store);
    })

    afterEach(() => {
        httpTesting.verify();
    })

    it('dispatches a store action on load', () => {
        const dispatchSpy = jest.spyOn(store, 'dispatch');
        
        service.loadPopularMovies().subscribe(data => {
            expect(dispatchSpy).toHaveBeenCalledTimes(1);
        });

        const req = httpTesting.expectOne('https://api.themoviedb.org/3/movie/popular?language=en-US&page=1');

        req.flush(LoadedMoviesResponse);
    })
```

## Testing a store the "old" way

```javascript
describe('advanced search store', () => {

    describe('selectors', () => {

        const initialState: AdvancedSearchState = {
            latestSearchTerm: "",
            loadedMovies: [...moviesMock],
            selectedMovieDetails: { ...movieMock }
        };


        it("should select ALL loaded movies", () => {
            const result = loadedMoviesSelector.projector(initialState);
            expect(result.length).toEqual(10);
        });


        it("should select details for a specific movie", () => {
            const result = specificMovieDetailsSelector.projector(initialState);
            expect(result).not.toBeNull();
            expect(result?.title).toEqual('The Great Adventure');
            expect(result?.id).toEqual(1);
        });
    })

  

    describe('reducers', () => {

        it('moviesLoaded reducer', () => {
            const initialState = { ...advancedSearchInitialState };

            const newState: AdvancedSearchState = {
                latestSearchTerm: initialState.latestSearchTerm,
                selectedMovieDetails: null,
                loadedMovies: moviesMock
            }

            const action = moviesLoaded({ movies: moviesMock });
            const state = advancedSearchReducer(initialState, action);

            expect(state).toEqual(newState);
            expect(state).not.toBe(initialState);
        });

  

        it('specificMovieDetailsLoaded reducer', () => {

            const initialState = { ...advancedSearchInitialState };

            const newState: AdvancedSearchState = {
                latestSearchTerm: initialState.latestSearchTerm,
                selectedMovieDetails: movieMock,
                loadedMovies: []
            }


            const action = specificMovieDetailsLoaded({ movie: movieMock });
            const state = advancedSearchReducer(initialState, action);

            expect(state).toEqual(newState);
            expect(state).not.toBe(initialState);
        });
    })
})
```