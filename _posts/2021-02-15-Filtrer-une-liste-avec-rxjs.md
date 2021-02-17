---
layout: post
category: [rxjs, angular]
keywords: [rxjs, angular]
---


[L'idée](#lidée) [Code](#code) [Démo](#démo) [Sources](#sources)

---
## [L'idée](#idea)
---

**La vue**

  - Un composant de list : ```ListComponent```
  - Un composant pour le champ de recherche : ```SearchInputComponent```
  - Un componsant de recherche : ```SearchComponent``` qui affichera le champ et la liste
  - Le module de recherche : ```SearchModule``` qui comprendra les composants cités au dessus

**Le flux**

  - On récupére au ```keyup``` la valeur entrée dans le champ du composant ```SearchInputComponent```
  - On filtre la liste dans ```SearchComponent```
  - On passe les données filtrées de la liste à l'enfant ```ListComponent```

**Le filtre**

C'est l'occasion d'utiliser l'observable ```combineLatest```, le ```BehaviorSubject``` et les opérateurs ```debounceTime``` et ```distinctUntilChanged```.<br>

On aura donc deux propriétés :
- ```searchFilter$``` qui recevra la valeur émise par le composant ```SearchComponent```
- ```list$``` qui sera utilisée pour afficher les résultats filtrés

---
## [Code](#code)
---

On va commencer par les <span class="info" title="Qui ne se charge que de l'affichage et d'émettre les évennements au composant parent">dumb components</span>. <br>
<br>

-***SearchListComponent***-

{% highlight typescript angular %}
import { Component, Input, ChangeDetectionStrategy } from "@angular/core";
import { IList } from "../search.interface";

@Component({
  selector: "search-list",
  template: `
    <ul>
      <li *ngFor="let item of list">
        {% raw %}{{ item.name }}{% endraw %}
      </li>
    </ul>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class SearchListComponent {
  @Input()
  public list: IList[] = [];
}
{% endhighlight %}

On commence donc par ```SearchListComponent```. Sans style, juste une liste à puce simple qui boucle sur le ```@Input()``` ```list``` et c'est tout.
<br>
<br>

-***SearchInputComponent***-

{% highlight typescript %}
import {
  Component,
  EventEmitter,
  Output,
  ChangeDetectionStrategy
} from "@angular/core";

@Component({
  selector: "search-input",
  template: `
    <input type="text" #inputSearch (keyup)="search(inputSearch.value)" />
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class SearchInputComponent {
  @Output()
  public onSearch: EventEmitter<string> = new EventEmitter<string>();

  public search(value: string): void {
    this.onSearch.emit(value);
  }
}
{% endhighlight %}

Puis ```SearchInputComponent```. Sans style non plus. A chaque ```keyup``` on récupère la valeur du champ ```input``` via sa référence ```#inputSearch``` puis on émet la valeur par le ```@Output()``` ```onSearch```.
<br>
<br>

-***SearchComponent***-

{% highlight typescript %}
import { Component } from "@angular/core";
import { debounceTime, distinctUntilChanged, map } from "rxjs/operators";
import { IList } from "./search.interface";
import { SearchService } from "./search.service";
import { BehaviorSubject, Observable, combineLatest } from "rxjs";

@Component({
  selector: "search",
  template: `
    <search-input (onSearch)="search($event)"></search-input>
    <search-list *ngIf="(list$ | async) as list" [list]="list"></search-list>
  `
})
export class SearchComponent {
  private readonly searchFilter$: BehaviorSubject<string> = new BehaviorSubject(
    ""
  );

  public list$: Observable<IList[]>;

  constructor(private readonly searchService: SearchService) {
    this.list$ = this.searchService.getNames();
  }

  public ngOnInit(): void {
    const search$: Observable<string> = this.searchFilter$.pipe(
      distinctUntilChanged(),
      debounceTime(300)
    );

    this.list$ = combineLatest([this.list$, search$]).pipe(
      map(([list, search]: [IList[], string]) =>
        this.filterByName(list, search)
      )
    );
  }

  public search(value: string): void {
    this.searchFilter$.next(value);
  }

  private filterByName(list: IList[], searchTerm: string): IList[] {
    if (searchTerm === "") return list;
    return list.filter(
      (item: IList) =>
        item.name.toLowerCase().indexOf(searchTerm.toLowerCase()) > -1
    );
  }
}
{% endhighlight %}

Notre <span class="info" title="Qui se charge de la logique, de l'injection des services et de l'état du composant mais pas de l'affichage">Smart component</span>.

Regardons le hook ```ngOnInit```, nous avons une constante ```list$``` qui récupère les valeurs retournées par la base de données (pour l'exemple, c'est une méthode du service ```SearchService``` qui retourne les données écrites en dur). <br>

Suit la constante ```search$``` qui récupère le flux du ```BehaviorSubject``` et effectue deux opérations :
- ```distinctUntilChanged``` : Compare la valeur précédente et la valeur courrante. Il fonctionne comme une condition, si la nouvelle valeur est différente de la valeur précédente alors il passe au prochain opérateur.
- ```debounceTime``` : Dans notre cas 300ms, on attend donc 300ms avant de passer au prochain opérateur. Si dans les 300ms la valeur change de nouveau, le compteur se remet à 0.

Ces opérateurs sont assez communs quand il s'agit d'écouter les évennements sur un champ texte. <br>

Avant de passer au ```combineLatest```, regardons la fonction ```search(value: string)```, elle est appellée chaque fois qu'un nouvel évennement est détecté, soit à chaque fois que le composant enfant ```SearchInputComponent``` notifie le parent d'un changement dans le champ texte. ```search(value: string)``` pousse dans le ```BehaviorSubject``` la nouvelle valeur, cette nouvelle valeur passe par les opérateurs que nous venons de décrire. <br>

```combineLatest``` est très utile dans notre cas car il attend que chaque observable émette au moins une fois une valeur puis prend la dernière valeur de chaque observable. Notre ```list$``` ne change pas mais à déjà émis une valeur, la liste initiale. Quant à ```search$```, il change de valeur à chaque changement dans le champ texte. <br>

Quand un changement est détecté, les valeurs des deux observables écoutés passent par l'opérateur ```map``` qui appelle la fonction ```filterByName(list: IList[], searchTerm: string)```, fonction qui si a ```searchTerm``` à vide retourne toute la liste, sinon effectue le tri et retourne les noms correspondants à la recherche.

**Points intéressants**

> 
  On utilise ```ChangeDetectionStrategy.onPush``` sur les <span class="info" title="Qui ne se charge que de l'affichage et d'émettre les évennements au composant parent">dumb components</span> qui fait en sorte que le composant ne se mette à jour que si au moins l'une de ses valeurs d'entrée a changé.
  <br>
  <br>
  ```combineLatest``` n'émet une valeur que si tous les observables passés en argument émette au moins une fois une valeur.
  <br>
  <br>
  Dans le template j'utilise ```list$ | async``` qui, entre autre, nous évite de ```unsubscribe``` explicitement l'observable ```list$```.

---
## [Démo](#demo)
---

<embed type="text/html" src="https://stackblitz.com/edit/filtrer-une-liste-avec-rxjs?ctl=1&embed=1&file=src/app/app.component.ts&theme=dark" width="100%" height="600">

<div class="embed-separator"></div>

**A voir également**

- <a>Filtrer n'importe quelle liste avec un @Pipe</a> *(bientôt)*
- <a>Différences entre Subject, BehaviorSubject et ReplaySubject</a> *(bientôt)*

---
## [Sources](#sources)
---

- [combineLatest](https://rxjs.dev/api/index/function/combineLatest)
- [ChangeDetectionStrategy.onPush](https://angular.io/api/core/ChangeDetectionStrategy)
- [Client side filtering with streams](https://blog.strongbrew.io/client-side-filtering-with-streams/)