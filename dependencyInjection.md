# Services und Dependency-Injection: lose Kopplung für Ihre Business-Logik

## Das Angular-Dependency-Injection-Framework

Das von Angular bereitgestellte Dependency-Injection-Framework basiert grundsätzlich auf der Idee der losen Kopplung. Anstatt die Abhängigkeiten, die Sie instanziieren wollen, im Injector selbst zu erzeugen, verwendet das Framework sogenannte Provider zur Konfiguration der Abhängigkeiten.

<strong>Einen Provider können Sie sich in diesem Zusammenhang als eine Regel vorstellen, die besagt, was passieren soll, wenn eine bestimmte Abhängigkeit benötigt wird.</strong>

### Injector- und Provider-Konfiguration: das Herz der DI

Die Konfiguration von Providern kann dabei entweder auf der Ebene eines NgModule (also z. B. auf Ebene des AppModule) oder auf Komponenten-Ebene erfolgen. Wird ein Provider innerhalb eines NgModule definiert, steht dieser allen Komponenten und Services innerhalb dieses Moduls zur Verfügung. Erfolgt die Konfiguration hingegen direkt in der Komponente, steht die Abhängigkeit der Komponente selbst und allen im Komponentenbaum darunter liegenden Komponenten zur Verfügung.

```ts
@NgModule({
  imports: [BrowserModule, HttpClientModule],
  providers: [{ provide: LoginService, useClass: LoginService }], // Provider werden hier definiert
  // providers: [LoginService] - vereinfachte Variante
  declarations: [AppComponent, LoginComponent],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

Der Ausdruck `{provide: LoginService, useClass: LoginService }` besagt somit:
_Wenn eine Komponente eine Abhängigkeit mit dem Token »LoginService« erfragt, gib eine Instanz der Klasse »LoginService« zurück._
Wenn eine Klasse einen Decorator besitzt, dann kann Typinformationen beim Kompilieren zu JS erhalten bleiben. In dieser Information befindet sich auch die angebe der Klassen, welche Injiziert werden.

```ts
LoginComponent = __decorate(
  [
    core_1.Component({
      selector: "ch-login",
      template: "...",
    }),
    __param(0, core_1.Inject(login_service_1.LoginService)), // Informationen für den DI-Mechanismus
    __metadata("design:paramtypes", [login_service_1.LoginService]),
  ],
  LoginComponent
);
```

Umgekehrt bedeutet die Verwendung dieses Tricks nun aber auch, dass nicht dekorierte Klassen auch nicht von dieser Vereinfachung profitieren können. Würde der LoginService nun – wie es sehr häufig der Fall ist – von weiteren Services abhängen,würde der soeben vorgestellte Mechanismus nicht mehr ohne Weiteres funktionieren: also immer verwenden.
Um die generierung von Metadaten zu erzwingen, stellt Angular den Decorator **@Injectable()** bereit.

```ts
import { Injectable } from '@angular/core';

@Injectable() // Decorator erzwingt generierung von Metadata
export class LoginService {
  constructor(http: HttpClient){
  ...
  }
}
```

## Weitere Provider-Formen

### useValue – Bereitstellen von Werten

Möchten Sie keine Klasseninstanzen, sondern einfache Werte bereitstellen, verwenden Sie die useValue-Eigenschaft.

```ts
{provide: 'greeting', useValue: 'Howdy'}
```

Diese Art der Injektion kann insbesondere dann hilfreich sein, wenn abhängig von der Umgebung unterschiedliche Konfigurationen oder Konstanten gebunden werden sollen.

### useFactory – Bereitstellen von Elementen über eine Factory

Beim Bereitstellen von Elementen über eine Factory wird dem Provider kein direkter Wert, sondern eine Funktion übergeben, die den bereitgestellten Wert berechnet.

```ts
export function generateRandomValue() {
  return Math.floor(Math.random() * 101);
}

@NgModule({
  providers: [
  {provide: 'random-value', useFactory: generateRandomValue}
  ...
})
```

Factory auf andere bereitgestellte Token zugreifen lassen:

```ts
export function getLoginService(useOAuth: boolean) {
  if (useOAuth) {
    return new OAuthLoginService();
  } else {
    return new LoginService();
  }
}

@NgModule ({
  providers: [
  {provide: 'ENABLE_OAUTH', useValue: true},
  {provide: LoginService, useFactory: getLoginService, deps: ['ENABLE_OAUTH']}
  ...
})
```

### useExisting – Bereitstellen über einen Alias

Das Bereitstellen über einen Alias ist die wohl exotischste Provider-Form.

```ts
@NgModule ({
providers: [
  {provide: LoginService,
  useClass: OAuthLoginService },
  {provide: 'currentLoginService', useExisting: LoginService },
  ...
})
```

In Ihrer Applikation können Sie ihn nun sowohl über die Anweisung _@Inject(LoginService) loginService_ als auch über die Anweisung _@Inject('currentLoginService') loginService_

### Injection-Tokens: kollisionsfreie Definition von DI-Schlüsseln

Das verwenden von Strings als Tokens für die Provider-Registrierung kann zum Problem werden, wenn man Fremdbibliotheken einbindet, da diese vielleicht den gleichen String als Token verwenden. Um diesem Problem vorzubeugen, bietet Angular Ihnen mit Injection-Tokens die Möglichkeit, Schlüssel auf Basis eines (Singleton-)Objekts zu definieren und somit den beschriebenen Kollisionen aus dem Weg zu gehen.

```ts
...tokens.ts
import {InjectionToken} from '@angular/core';
export const RANDOM_VALUE = new InjectionToken<number>('random-value');
```

```ts
...app.module.ts
import {RANDOM_VALUE} from './app-tokens.ts';

@NgModule({
providers: [
  ...
  {provide: RANDOM_VALUE, useFactory: generateRandomValue}]
})
export class LoginComponent implements OnInit {}
```

## Der hierarchische Injector-Baum: volle Flexibilität bei der Definition Ihrer Abhängigkeiten Sie kennen jetzt alle Grundlagen der Dependency-Injection-API und wissen, wie

### Registrierung von globalen Services

Wenn sie eine Service global anbieten wollen, dann erfolgt die Registrierung am RootInjector, also in _app.module.ts_.

### Registrieren von komponentenbezogenen Services

Um einen Service an nur eine Komponente zu registrieren erfolgt die Registrierung von Abhängigkeiten in diesem Fall über die providers-Eigenschaft des @Component-Decorators. Zur Erstellung von wiederverwendbaren Services (MusicSearchService, LibrarySearchService) kann die Verwendung der abstrakten Klasse genutzt werden.

```ts
export abstract class SearchService {
  abstract search(keyword: string): any[];
}
```

Die abstrakte Klasse definiert hier lediglich die Schnittstelle, die durch eine Implementierung bereitgestellt werden muss. Im nächsten Schritt lassen Sie nun den MusicSearchService diese Klasse implementieren:

```ts
class MusicSearchService implements SearchService {
  constructor(http: HttpClient) {}
  search(keyword: string): any[] {
    // ... lade Daten aus dem Music-Backend
  }
}
```

In der MusicLibraryComponent erstellen Sie einen Provider, der dafür sorgt, dass bei der Injection eines SearchService eine Instanz des MusicSearchService bereitgestellt wird.

```ts
...music-library.component.ts
@Component({
  providers: [{provide: SearchService, useClass: MusicSearchService}],
}
...
```

Die Implementierung der SearchBarComponent kann dadurch jetzt völlig ohne Annahmen über den aktuellen fachlichen Kontext erfolgen:

```ts
@Component({
  selector: "ch-search-bar",
  template: ` <label>Suche</label>
    <input type="text" #query (keyup.enter)="search(query.value)" />
    <button (click)="search(query.value)">Suchen</button>`,
})
export class SearchBarComponent {
  results: any[];
  constructor(private searchService: SearchService) {}
  search(searchQuery) {
    this.results = this.searchService.search(searchQuery);
  }
}
```

## Treeshakable-Providers: der DI-Mechanimus auf den Kopf gestellt

Mit Treeshakable-Providers steht Ihnen seit Angular 6 eine weitere interessante Option zur Registrierung eines Injectable-Service zu Verfügung. So bietet Ihnen der @Injectable-Decorator, zusätzlich die Möglichkeit, zu bestimmen, in welchem NgModule der jeweilige Service bereitgestellt (provided) werden soll. In diesem Fall können Sie anschließend darauf verzichten, den Provider explizit im providers-Array des betreffenden Moduls anzugeben.

```ts
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(private http: HttpClient) {}
...
}
```

## Sichtbarkeit und Lookup von Dependencys

### Sichtbarkeit von Providern beschränken

Damit nur reguläre Kinder und nicht auch die Content-Children ein registrierter Service zur Verfügung steht kann man _viewProviders_ im Decorator benutzen.

```ts
@Component({
  selector : 'ch-main-view',
  viewProviders: [{provide: SearchService,
  useClass: GlobalSearchService}]
})
```

### Den Lookup von Abhängigkeiten beeinflussen

Oft ist es hilfreich, den Lookup der Abhängigkeiten gezielt zu steuern. So sorgt der @Inject-Decorator ohne weitere Konfiguration immer dafür, dass zu-
nächst innerhalb des eigenen Injectors nach der Abhängigkeit gesucht wird. Wird diese dort nicht gefunden, läuft die Suche den Dependency-Tree bis zum Root-Injector hinauf. Wird immer noch keine Abhängigkeit gefunden, löst das DI-Framework einen Fehler aus. Falls dieses Verhalten so nicht gewünscht ist, bietet Angular Ihnen einige zusätzliche Möglichkeiten, um den Lookup zu beeinflussen.

#### @Optional – optionale Abhängigkeiten definieren

Nicht immer ist eine Abhängigkeit zwangsläufig notwendig, damit die Applikation funktioniert. Möchten Sie eine Abhängigkeit als optional definieren, können Sie dies über den @Optional-Decorator tun.

```ts
import {Optional} from '@angular/core';
constructor(@Optional() cacheService: CacheService) {
  if (cacheService) {
  ...
  }
}
```

#### @Host – nur innerhalb des aktuellen Hosts suchen

Mithilfe des @Host-Decorators können Sie dem Injector mitteilen, dass der Lookup den Dependency-Tree maximal bis zum nächsten Host-Element hinauflaufen soll. Wollten Sie beispielsweise für den Lookup des SearchService in der SearchBarComponent festlegen, dass innerhalb des jeweiligen Hosts ein passender SearchService registriert sein muss, so könnten Sie dies mithilfe der folgenden Anweisung tun:

```ts
import {Host} from '@angular/core';
...
constructor(@Host() searchService: SearchService) {}
```

Dies kann beispielsweise immer dann sinnvoll sein, wenn Ihre Komponente oder Ihr Service eine eigene Instanz eines Service benötigt und Sie verhindern möchten, dass aus Versehen ein globaler Provider verwendet wird.

#### @SkipSelf – den eigenen Injector auslassen

Mithilfe des @SkipSelf-Decorators können Sie festlegen, dass der Lookup für die entsprechende Abhängigkeit direkt beim Parent-Injector beginnen soll. Der eigene Injector wird in diesem Fall ausgelassen. Dies kann beispielsweise bei der Implementierung von rekursiven Strukturen notwendig sein.

```ts
...directory.component.ts
import {..., Optional, SkipSelf} from '@angular/core';
@Component({
  selector: 'ch-directory',
  template: `
    <div>{{name}}</div>
    <div class="child">
    <ng-content></ng-content>
    </div>
  `
})
export class DirectoryComponent implements OnInit {
  @Input() name: string;
    constructor(@Optional() @SkipSelf() private parent: DirectoryComponent) {
  }

  ngOnInit() {
    const parent = this.parent ? this.parent.name : 'null';
    console.log('Name: ' + this.name + ' Parent: ' + parent);
  }
}
```

```html
<ch-directory name="root">
  <ch-directory name="child1"></ch-directory>
  <ch-directory name="child2"></ch-directory>
</ch-directory>
```

Ein Weglassen des @SkipSelf-Decorators hätte in diesem Fall hingegen dazu geführt, dass das DI-Framework zunächst im eigenen Injector nach einer Directory-Abhängigkeit fragt. Dort würde die Komponente selbst als Abhängigkeit gefunden werden. Der Versuch, diese zu injizieren, würde mit einer CyclicDependencyException fehlschlagen.

#### @Self – nur innerhalb des eigenen Injectors suchen

In gewissen Fällen kann es notwendig sein, sicherzustellen, dass eine Abhängigkeit lediglich im aktuellen Injector gesucht wird. So besteht ein gängiger Anwendungsfall darin, das Verhalten einer Direktive nur dann zu verändern, falls eine andere Direktive an diesem Element vorhanden ist.

```ts
@Directive({
  selector: "[chAlert]",
})
export class AlertDirective {
  constructor(private el: ElementRef) {
    this.el.nativeElement.style.color = "red";
    this.el.nativeElement.style["font-weight"] = "BOLD";
  }
}
```

Ein mögliches Szenario könnte nun darin bestehen, zusätzlich eine Border-Direktive zu implementieren. Diese Direktive soll einen Rahmen um das dekorierte Element zeichnen. Abhängig davon, ob am gleichen Element zusätzlich eine AlertDirective definiert wurde, soll dieser Rahmen entweder einen Pixel oder drei Pixel breit sein.

```ts
@Directive({
  selector: "[chBorder]",
})
export class BorderDirective {
  constructor(
    private el: ElementRef,
    @Self() @Optional() alert: AlertDirective
  ) {
    const borderWidth = alert ? "3px" : "1px";
    this.el.nativeElement.style.border = "solid " + borderWidth;
  }
}
```
