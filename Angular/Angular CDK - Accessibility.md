
# CdkTrapFocus

This directive is useful when you want to prevent focus from moving way from an element or region using the *keyboard* (this only traps focus for keyboard interactions, you can still click away with the mouse)

There are 2 ways you can apply this:

`CdkFocusTrap`

```html
<ul tabindex="0" cdkTrapFocus>
	<li tabindex="0">1</li>
	<li tabindex="0">2</li>
	<li tabindex="0">3</li>
	<li tabindex="0">4</li>
	<li tabindex="0">5</li>
</ul>
```

If you do it like this, your keyboard navigation will be stuck there in the `ul` element

`Using Regions`

```html
<ul tabindex="0" cdkTrapFocus="">
	<li tabindex="0" cdkFocusRegionStart>1</li>
	<li tabindex="0">2</li>
	<li tabindex="0">3</li>
	<li tabindex="0">4</li>
	<li tabindex="0" cdkFocusRegionEnd>5</li>
</ul>
```

This works the same as above with the only exception that you now are able to tab between the `li` but you cannot exit them (once you get to 5 you go back to 1). This needs the `cdkTrapFocus` directive to work.