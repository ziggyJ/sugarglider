---
title: How To Use Custom Renderer To Avoid DOM Creation
date: 2017-02-02 17:04:14
tags:
---

Angular 2 is quite powerful. It has execellent Change Detections, full Component support. 
So a lot of developers use Angular 2 to create Charts like [ngx-charts](https://github.com/swimlane/ngx-charts) 
or manage Google Maps Component like [angular2-google-maps](https://github.com/SebastianM/angular2-google-maps)

In a project, I built a Hybrid app using ionic 2 that contains a big Map component using [Google Maps Cordova Plugin](https://github.com/mapsplugin/cordova-plugin-googlemaps)
Inspired by Angular2-google-maps project, I also wrap the Google Maps Cordova plugin in a Angular 2 components and let Angular 2 to manage the Markers Polygons Polylines.

The Map component is very complicate, sometimes it contains thousands of markers. It is a bit slow to load the all markers. I did some profiling using Chrome Dev Tool and find a big part of time is used on DOM operations.
As we know DOM is not fast. Angular 2 will use DomRenderer by default, and it will create a dom element for each component or directive. In my case, I just use Angular 2 to manage the native Google maps components,
Actually I do not need any DOM.

Victor Savkin wrote an execellent [Article about the Custom Renderer](https://blog.nrwl.io/experiments-with-angular-renderers-c5f647d4fd9e#.38u9i09l8) and in that article, 
he implemented an in memory renderers. The implementation needs to implement a custom RootRenderer, and the custom RootRenderer will provide each component a InMemRenderer
then component will use that InMemRenderer to render the component template. It will just create element in memory without creating any DOM. It is quite cool.

But we can not just use InMemRootRenderer, because our app has other part besides maps. Those parts will need DOM. Can we just use InMemRenderer to render Map components, 
but use DomRenderer to render other components?

Actually we can. The RootRenderer use renderComponent function to decide what Renderer the component will use. The function will accept an RenderComponentType
as parameter and return an Renderer

Let's take a look RenderComponentType definition

```typescript
export declare class RenderComponentType {
    id: string;
    templateUrl: string;
    slotCount: number;
    encapsulation: ViewEncapsulation;
    styles: Array<string | any[]>;
    animations: {
        [key: string]: Function;
    };
    constructor(id: string, templateUrl: string, slotCount: number, encapsulation: ViewEncapsulation, styles: Array<string | any[]>, animations: {
        [key: string]: Function;
    });
}
```
It does not have the Component Name or Component Class!!!. But luckily it has the styles property, so we can set a special style on the components that need a custom renderer
and the RootRenderer can use that information to return a custom renderer.

This the my custom root renderer.

```typescript
@Injectable()
export class HybridRootRenderer extends DomRootRenderer {
  public roots: any[] = [];

  constructor(@Inject(DOCUMENT) _document: any, _eventManager: EventManager,
              sharedStylesHost: DomSharedStylesHost, animate: AnimationDriver) {
    super(_document, _eventManager, sharedStylesHost, animate);
  }

  public renderComponent(componentProto: RenderComponentType): Renderer {
    let renderer: Renderer = this.registeredComponents.get(componentProto.id);
    if (!renderer) {
      if (componentProto.styles.length === 1 && componentProto.styles[0] === 'InMem') {
        // console.log('InMem', componentProto);
        renderer = new InMemoryRenderer(this.roots);
      } else {
        renderer = new DomRenderer(
          this, componentProto, this.animationDriver);
      }
      this.registeredComponents.set(componentProto.id, <any>renderer);
    }
    return renderer;
  }
}
```

and this is the component which will use the custom render


```typescript
@Component({
  selector: 'aw-mob-marker',
  template: `<aw-marker
      [latitude]="mobMarker?.markerLocation?.latitude"
      [longitude]="mobMarker?.markerLocation?.longitude"
      draggable="true"
      [visible]="mobMarker?.visible"
      [opacity]="mobMarker?.opacity"
      [id]="mobMarker?.markerLocation?.id"
      [icon]="icon"
      [zIndex]="mobMarker?.zIndex"
      (click)="_markerClicked($event)"
      (markerDragStart)="_markerDragStart($event)"
      (markerDrag)="_markerDrag($event)"
      (markerDragEnd)="_markerDragEnd($event)"
    ></aw-marker>`,
  styles: ['InMem'],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AwMobMarkerComponent implements OnInit, OnChanges, OnDestroy {
  ...
```

As the result, I can see the all Dom operations are gone on the map components. 