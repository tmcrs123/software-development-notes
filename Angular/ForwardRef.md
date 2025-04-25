`ForwardRef` is a way of breaking circular dependencies; it's a way of deferring dependency resolution . Essentially it's you telling Angular "don't resolve this just now, resolve only when you need so that you can be certain it's defined".

This is not exclusive to services. You can apply this to a circular dependency of anything, be it services, components or whatever else.

The classic use case is when implementing the CVA

```javascript
@Component({

  selector: 'rating',

  imports: [],

  providers: [{ provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => RatingComponent), multi: true }],

  templateUrl: './rating.component.html',

  styleUrl: './rating.component.scss'

})

export class RatingComponent implements ControlValueAccessor {}
```

If you don't use the `forwardRef` here, when you get to `useExisting` the `RatingComponent` class won't be defined yet. That's why you need `forwardRef` here.