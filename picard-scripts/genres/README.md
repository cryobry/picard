# Standardize genres in MusicBrainz Picard

Map MusicBrainz genres to consistent labels and write the result to every track in a matched release. This setup uses Picard 3.0.5 and the files below.

| File | Where it goes |
| --- | --- |
| [genre-mapper-rules.txt](./genre-mapper-rules.txt) | Paste into **Options → Plugins → Genre Mapper → Replacement Pairs**. |
| [Configure Genres.ptsp](./Configure%20Genres.ptsp) | Import in **Options → Scripting**. Edit its settings to change buckets, aliases, and preferences. |
| [Standardize Genres.ptsp](./Standardize%20Genres.ptsp) | Import in **Options → Scripting** and run after **Configure Genres**. |
| [Clear Artist Genre Cache.ptsp](./Clear%20Artist%20Genre%20Cache.ptsp) | Optional: run on selected releases before refreshing to clear cached artist results. |

## 1. Install plugins

Open **Options → Options… → Plugins → Install Plugin…** and install these from the Picard plugin registry. Confirm that they are enabled. On macOS, use **MusicBrainz Picard → Preferences…**. [Plugin installation](https://picard-docs.musicbrainz.org/en/v3.0/config/options_plugins.html)

| Plugin | What it supplies |
| --- | --- |
| **Genre Mapper** | Applies `genre-mapper` to each track's loaded genres. [Guide](https://picard-plugins-user-guides.readthedocs.io/en/latest/genre_mapper/user_guide.html) |
| **Persistent Variables** | Shares votes and results across a release and holds the optional artist cache. No extra settings are needed. [Guide](https://picard-plugins-user-guides.readthedocs.io/en/latest/persistent_variables/user_guide.html) |
| **Additional Artists Variables** | Provides `%_artists_album_primary_tags%` for album-artist matching. Use a version that exposes this variable. [Guide](https://picard-plugins-user-guides.readthedocs.io/en/latest/additional_artists_variables/user_guide.html#album-variables) |

## 2. Configure inputs and replacement rules

1. In **Options → Metadata → Genres**, enable **Use genres from MusicBrainz**. Start with **Maximum number of genres** at **3** and leave **Join multiple genres with** blank. If too few genres arrive, check **Minimal genre usage** and the filters. [Genre options](https://picard-docs.musicbrainz.org/en/v3.0/config/options_genres.html)
2. Optionally enable **Fall back to album’s artists genres**. Leave **Only use my genres** and **Use folksonomy tags as genre** off to start from MusicBrainz's recognized genres.
3. In **Options → Plugins → Genre Mapper**, paste the full plain-text contents of `genre-mapper` into **Replacement Pairs**. Enable **Enable genre mapping**, **Match tests are entered as regular expressions**, and **Apply only the first matching replacement**. Leave **Case-sensitive matching** off. [Genre Mapper options](https://picard-plugins-user-guides.readthedocs.io/en/latest/genre_mapper/user_guide.html#option-settings)

## 3. Import and run scripts

1. In **Options → Scripting**, enable **Enable Tagger Script(s)** and import both `.ptsp` files. Disable older copies when updating.
2. Enable both scripts and put **Configure Genres** above **Standardize Genres**. Picard runs the plugins first, then Configure Genres across the tracks, then Standardize Genres. [Scripting](https://picard-docs.musicbrainz.org/en/v3.0/config/options_scripting.html) · [Processing order](https://picard-docs.musicbrainz.org/en/v3.0/extending/processing.html)
3. Match a release. After changing artist genre rules, run **Clear Artist Genre Cache** on the selected release, then use **Refresh** so the new rules are applied. Check **Original Value** and **New Value** for **Genre** before saving. [Metadata pane](https://picard-docs.musicbrainz.org/en/v3.0/getting_started/metadata_pane.html)

### How the final genre list is chosen

- Configure Genres splits crossover outputs such as `Dance; Pop`, trims them, and counts each genre once per track across all discs.
- For a single identified album artist, Standardize Genres prefers recognized artist tags, then the session cache if enabled, then release votes. Various Artists and multi-artist releases use release votes.
- A release-vote genre qualifies with at least half the leader's votes. Results are ordered by count, then alphabetically. Redundant parents are removed, the limit is applied, and release-only labels are appended.
- `Other` is a last resort. If a matched file supplies the only genre, its existing values are kept.

## 4. Edit Configure Genres

Edit the settings at the top of **Configure Genres**. All script configuration is there; you normally do not need to edit **Standardize Genres**. Leave the collection logic below the settings unchanged unless you are modifying the script.

| Setting | Default and how to use it |
| --- | --- |
| `_ag_cache_artist` | `1` caches results by artist MBID when no recognized artist tags are available; use `0` to use each release's votes. Restart Picard to clear the cache. |
| `_ag_limit` | `3` limits the final list after deduplication and parent removal. This is separate from Picard's input maximum. |
| `_ag_album_only` | `Holiday` is appended after the limit and excluded from the artist cache. |
| `_ag_ignore` | `Other` is excluded from matching and ordinary votes, then used as a last resort. |
| `_ag_match_suffix` | `1` removes a parent at the end of a label: `Indie Rock` removes `Rock`. |
| `_ag_prefix` | Parents listed here are removed at the start of a label: `Indie Pop` removes `Indie`. |
| `_ag_children_<Parent>` | Lists children that suppress a parent, such as `Synthpop; Vocal Pop` for `Pop`. |
| `_ag_buckets` | Output labels eligible for direct artist matching. Match the mapper's spelling and capitalization. |
| `_ag_alias_<key>` | Maps a normalized raw artist tag to output labels. Keys use lowercase with spaces and hyphens removed; aliases override direct bucket matches. |

## 5. Keep the mapper and artist aliases in sync

Genre Mapper processes track `genre` values. Standardize Genres separately matches album-artist tags using `_ag_buckets` and `_ag_alias_*`; both paths can affect the result.

- Use identical spelling and capitalization in mapper replacements, `_ag_buckets`, aliases, and parent rules. Put each crossover output in `_ag_buckets` separately.
- Mapper rows use `regex=replacement`. Put specific rules first and keep `^.*$=Other` last. Lines beginning with `=` are comments. Add new input buckets to the mapper's **Existing bucket names** rule or give them an exact match.
- Add aliases only when a raw artist tag should map to a different bucket. Keys are normalized as described above; for example, `traditional pop` uses `_ag_alias_traditionalpop`.
- `Holiday` and `Other` are intentionally handled by `_ag_album_only` and `_ag_ignore`. Test both a release genre and an album-artist tag, then refresh the release.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| A genre becomes a broad label or `Other` | Check the first matching mapper rule and the normalized artist alias, if applicable. |
| Artist overrides do not apply | Confirm Additional Artists Variables exposes `%_artists_album_primary_tags%`, the release has one album artist, and the raw tag appears in `ag_artist_tags`. |
| No new result appears | Confirm both scripts are enabled and ordered correctly. In **Plugins → View persistent variables**, check `ag_tracks`, `ag_labels`, and `ag_final`. |
| Changes have no effect | Refresh the release. Restart Picard to clear the artist cache, or set `_ag_cache_artist` to `0` while testing. |
| Multiple genres arrive as one value | Leave **Join multiple genres with** blank and repaste the plain-text mapper rules with regex mode enabled. |

[Persistent Variables inspection and cache behavior](https://picard-plugins-user-guides.readthedocs.io/en/latest/persistent_variables/user_guide.html)
