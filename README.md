# Tasks app: dwa małe serwisy

API listy zadań, które wywołuje małą funkcję bez stanu do porządkowania tytułów zadań.

| Serwis | Folder | Port | Co robi |
|---|---|---|---|
| tasks-api | `services/tasks-api` | 5000 | CRUD dla zadań. Przy tworzeniu zadania wywołuje `title-normalizer`. |
| title-normalizer | `services/title-normalizer` | 5001 | Jedna funkcja: `POST /invoke` obcina tytuł, zwija spacje i zamienia pierwszą literę na wielką. |

Jeśli `title-normalizer` nie działa, `tasks-api` nadal tworzy zadanie z surowym tytułem i `"normalized": false`.

## Endpointy

tasks-api:

- `GET /health`
- `GET /tasks`
- `POST /tasks` z `{"title": "..."}`: 201, albo 400 dla pustego tytułu lub treści, która nie jest JSON-em
- `GET /tasks/<id>`: 200 lub 404
- `DELETE /tasks/<id>`: 204 lub 404

title-normalizer:

- `GET /health`
- `POST /invoke` z `{"title": "  buy   milk "}` zwraca `{"title": "Buy milk"}`; 400, jeśli `title` nie jest tekstem

## Uruchomienie (dwa terminale)

Każdy serwis ma własny plik `requirements.txt`.

```bash
# terminal 1
cd services/title-normalizer
pip install -r requirements.txt
flask --app app:create_app run --port 5001

# terminal 2
cd services/tasks-api
pip install -r requirements.txt
NORMALIZER_URL=http://localhost:5001 flask --app app:create_app run --port 5000
```

Wypróbuj:

```bash
curl -X POST localhost:5000/tasks -H 'Content-Type: application/json' -d '{"title": "  buy   milk "}'
curl localhost:5000/tasks
```

Zatrzymaj terminal 1 i utwórz kolejne zadanie: nadal działa, z `"normalized": false`.

## Testy

Uruchom w folderze każdego serwisu:

```bash
pytest
```

## Firewall w Codespaces

Jeśli kontenery nie mogą się ze sobą połączyć w twoim Codespace, uruchom to jednorazowo. Nowy Codespace wymaga tego od nowa. Druga linia powinna wypisać `-P FORWARD ACCEPT`:

```bash
sudo iptables-legacy -P FORWARD ACCEPT
sudo iptables-legacy -S FORWARD | head -1
```

## Co obejmuje każde spotkanie

Repo rośnie spotkanie po spotkaniu. Najpierw pipeline (`.github/workflows/ci.yml`), potem kontenery, infrastruktura i monitoring. Znajdź poniżej swoje spotkanie, żeby zobaczyć, czego dotyczy. Nic tutaj nie jest pracą domową, a późniejsze tematy są planowane: kolejność i szczegóły mogą się zmienić.

### Spotkanie 1: pierwszy CI

Workflow, który uruchamia się przy każdym pushu: pobiera kod, ustawia Pythona, instaluje zależności, uruchamia `pytest`.

Zadania: zepsuj test na branchu i przeczytaj czerwony przebieg. Znajdź, który krok się nie powiódł i dlaczego.

### Spotkanie 2: CI w szczegółach

Ten sam workflow dla obu serwisów i kilku wersji Pythona (macierz). Wynik testów jest zapisany jako artefakt. Skan bezpieczeństwa Bandit tworzy raport HTML, również zapisany jako artefakt. `continue-on-error` pozwala jobowi iść dalej, gdy Bandit coś znajdzie.

Zadania: przeczytaj części o macierzy i `upload-artifact` w [składni workflow](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax). Pobierz artefakt ze strony przebiegu i otwórz go.

### Spotkanie 3: CD z GitHub Pages

Job `deploy` czeka na `run_tests` (`needs`), pobiera dwa raporty Bandita, buduje `index.html` i publikuje całość przez GitHub Pages. Job `generate_api_docs` tworzy dokumentację API obu serwisów przez pdoc, a strona główna linkuje do niej. Deploy uruchamia się tylko po scaleniu pull requesta: trigger `pull_request` ma typy aktywności `opened`, `synchronize` i `closed`, a `deploy` sprawdza pole `merged`. Pages wymaga publicznego repo na darmowym koncie, a to, co publikujesz, jest publiczne.

Dwóch ustawień nie ma w pliku. Ustaw je we własnym repo:

- Settings > Pages > Source: GitHub Actions (repo musi być publiczne).
- Settings > Environments > github-pages > Deployment branches and tags: dodaj regułę `refs/pull/*/merge`. Bez niej job `deploy` scalonego pull requesta zostanie odrzucony z komunikatem: Branch "refs/pull/N/merge" is not allowed to deploy to github-pages due to environment protection rules.

Jeśli nie było cię na tym spotkaniu: zastąp `.github/workflows/ci.yml` plikiem z tego repo, usuń `.github/workflows/ci_livecode.yml`, jeśli go masz, i ustaw dwa ustawienia powyżej. Nic z tego nie jest pracą domową.

Dokumentacja: [Using custom workflows with GitHub Pages](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages), [events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows).

Zadania:

1. Automatycznie generowana dokumentacja API: dodaj dowolne narzędzie dokumentujące kod Flaska, uruchom polecenie tworzące HTML, wgraj go, opublikuj na swojej stronie i podlinkuj ze strony głównej. Nie zmieniaj `services/`. [pdoc](https://pdoc.dev/) to jedna z łatwych opcji.
2. Deploy tylko po scaleniu pull requesta (sprawdź typy aktywności `pull_request` i pole `merged`).
3. Przeczytaj cały swój `ci.yml` i wyjaśnij sobie każdy job.

### Spotkanie 4: obrazy i kontenery Dockera

Dockerfile opisuje obraz: obraz bazowy plus kroki, które dodają twój serwis. Budujesz obraz, uruchamiasz go jako kontener i docierasz do niego przez port.

Jeśli nie było cię na tym spotkaniu: skopiuj te cztery pliki z tego repo do swojego, każdy w całości: `services/tasks-api/Dockerfile`, `services/tasks-api/.dockerignore`, `services/title-normalizer/Dockerfile`, `services/title-normalizer/.dockerignore`. Nie ma ustawień do zmiany, a `ci.yml` się nie zmienia. Nic z tego nie jest pracą domową.

Dokumentacja: [Dockerfile reference](https://docs.docker.com/reference/dockerfile/), [Building best practices](https://docs.docker.com/build/building/best-practices/), [Forwarding ports in your codespace](https://docs.github.com/en/codespaces/developing-in-a-codespace/forwarding-ports-in-your-codespace).

Zadania:

1. Dockerfile dla `tasks-api`: zbuduj obraz, uruchom go, potem `curl` na `/health` i `POST /tasks`. Nie zmieniaj `services/`.
2. To samo dla `title-normalizer`.
3. Plik `.dockerignore` dla każdego serwisu.
4. Uruchom kontener jako użytkownik, który nie jest rootem.

### Plany na później

- Docker Compose i publikowanie obrazów: oba serwisy z jednego pliku `compose.yaml`, docierające do siebie po nazwie serwisu, job CI `build_images`, publikowanie obrazu w rejestrze (`ghcr.io`). Dokumentacja: [Compose file reference](https://docs.docker.com/reference/compose-file/), [Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry).
- Ochrona `main` i skanowanie obrazów: ruleset z wymaganymi pull requestami i checkami, job `scan_images` z Trivy, przypinanie akcji po commicie. Dokumentacja: [About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets), [Trivy Action README](https://github.com/aquasecurity/trivy-action#readme).
- Porządek w zależnościach i repo: Dependabot, obraz bazowy przypięty po digeście, `CODEOWNERS`, szablon pull requesta, liniowa historia. Dokumentacja: [Dependabot options reference](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference), [About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners).
- Historia gita: branche, cherry-pick, squash merge i rebase. Dokumentacja: [Git Branching - Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing).
- Monitoring: metryki z aplikacji, Prometheus, Grafana. Dokumentacja: [Prometheus overview](https://prometheus.io/docs/introduction/overview/).
