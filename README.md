# hostido-php-deploy

Wspólny workflow GitHub Actions (`workflow_call`), który wgrywa stronę PHP na hosting Hostido przez SSH i rsync.

Kolejność kroków:

1. checkout repozytorium,
2. `composer install --no-dev` (jeśli jest `composer.json` i ustawione `php_version`),
3. `npm ci` + `build_command` (jeśli ustawione `build_command`, wymaga `package-lock.json`),
4. kontrola katalogu docelowego (patrz niżej),
5. `rsync -az --delete` do `remote_path`,
6. zapis sekretu `dotenv` do `<remote_path>/.env` (prawa 600),
7. GET na `post_deploy_url` (np. czyszczenie cache).

## Użycie

```yaml
jobs:
  deploy:
    uses: mich418/hostido-php-deploy/.github/workflows/deploy.yml@v1
    with:
      remote_path: domains/example.com/app   # ścieżka względem katalogu domowego albo absolutna, bez "~"
      build_command: npm run build           # opcjonalnie
      exclude: |                             # opcjonalnie, dodatkowe wzorce rsync
        uploads/
    secrets:
      ssh_host: ${{ secrets.HOSTIDO_SSH_HOST }}
      ssh_port: ${{ secrets.HOSTIDO_SSH_PORT }}
      ssh_user: ${{ secrets.HOSTIDO_SSH_USER }}
      ssh_private_key: ${{ secrets.HOSTIDO_SSH_KEY }}
      dotenv: |
        API_TOKEN=${{ secrets.API_TOKEN }}
        OTHER=value
      post_deploy_url: ${{ secrets.CACHE_CLEAR_URL }}
```

### Inputs

| Nazwa | Domyślnie | Opis |
|---|---|---|
| `remote_path` | — (wymagane) | Katalog docelowy na serwerze, tworzony jeśli go brak |
| `php_version` | `8.3` | Wersja PHP dla Composera; pusty string pomija Composera |
| `node_version` | `22` | Wersja Node.js dla `build_command` |
| `build_command` | `''` | Komenda budująca po `npm ci` |
| `exclude` | `''` | Dodatkowe wzorce rsync, po jednym w linii |

### Secrets

| Nazwa | Wymagany | Opis |
|---|---|---|
| `ssh_host`, `ssh_user`, `ssh_private_key` | tak | Dane SSH z panelu Hostido |
| `ssh_port` | nie | Domyślnie 22 |
| `ssh_known_hosts` | nie | Linie `known_hosts`; bez tego używany jest `ssh-keyscan` |
| `dotenv` | nie | Zawartość pliku `.env` |
| `post_deploy_url` | nie | Adres wywoływany po deployu |

### Ochrona przed skasowaniem cudzych plików

`rsync --delete` usuwa z `remote_path` wszystko, czego nie ma w repozytorium. Dlatego workflow:

- odrzuca `remote_path` pusty, `/`, `.`, zaczynający się od `~` albo zawierający `..`,
- wgrywa pliki tylko do katalogu, który nie istnieje, jest pusty albo zawiera plik `.hostido-deploy` (tworzony przez workflow po pierwszym udanym deployu).

Jeśli chcesz celowo użyć istniejącego katalogu, opróżnij go albo utwórz w nim `.hostido-deploy` ręcznie.

### Co nie jest nadpisywane ani usuwane na serwerze

`.hostido-deploy`, `.git/`, `.github/`, `tests/`, `node_modules/`, `var/`, `.env`, `.env.*`, `phpunit.xml*`, `.phpunit.cache/` oraz wzorce z `exclude`. `var/` to miejsce na cache i dane runtime aplikacji, a `.env` zapisuje osobny krok workflow.

## Przygotowanie Hostido

1. W panelu Hostido włącz dostęp SSH (jeśli go nie ma, poproś support) i sprawdź host, port i nazwę użytkownika.
2. Wygeneruj klucz tylko do deployu: `ssh-keygen -t ed25519 -C github-deploy -f hostido_deploy -N ""`.
3. Dodaj `hostido_deploy.pub` w panelu (Klucze SSH) albo do `~/.ssh/authorized_keys` na serwerze.
4. Zawartość `hostido_deploy` (klucz prywatny) wpisz jako sekret repozytorium wywołującego workflow.
5. Katalog dokumentów domeny ustaw na publiczny podkatalog aplikacji (np. `<remote_path>/public`), żeby `.env`, `vendor/` i `var/` nie były dostępne z sieci.

## Dostęp z innych prywatnych repozytoriów

To repozytorium jest prywatne. Żeby inne repozytoria tego samego właściciela mogły z niego korzystać:
Settings → Actions → General → Access → „Accessible from repositories owned by the user 'mich418'”.

## Wersjonowanie

Repozytoria wywołujące używają tagu `v1`. Po zmianach zgodnych wstecz przesuń tag:

```bash
git tag -f v1 && git push -f origin v1
```
