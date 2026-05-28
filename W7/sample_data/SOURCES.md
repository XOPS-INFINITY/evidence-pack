# StudyBot sample data — Sources & Citations

## Provenance

The 5 `wiki_*.txt` sample files in this directory are **real Wikipedia articles** sourced from HuggingFace via:

```bash
python3 tooling/fetch_w7_datasets.py --app studybot
```

| File | Wikipedia article | URL |
|------|-------------------|-----|
| `wiki_01_computer.txt` | Computer | https://simple.wikipedia.org/wiki/Computer |
| `wiki_02_internet.txt` | Internet | https://simple.wikipedia.org/wiki/Internet |
| `wiki_03_mathematics.txt` | Mathematics | https://simple.wikipedia.org/wiki/Mathematics |
| `wiki_04_photosynthesis.txt` | Photosynthesis | https://simple.wikipedia.org/wiki/Photosynthesis |
| `wiki_05_energy.txt` | Energy | https://simple.wikipedia.org/wiki/Energy |

## Generated lecture PDF

| File | Description | License |
|------|-------------|---------|
| `ai_study_buddy_40_slide_lecture.pdf` | Synthetic 40-slide StudyBot lecture deck for upload, summary, quiz, dashboard, citation, chunking, and retrieval-quality tests. Includes table-like rows, figure caption text, equation text, code-like text, and explicit `Slide N` markers. | Project sample data |

This PDF is generated sample content, not copied from an external source. It is intended for local demos of the user story: upload a 40-slide lecture PDF and receive a one-page summary with five testable concepts.

## HuggingFace dataset

- **Name:** `wikimedia/wikipedia`
- **Config:** `20231101.simple` (Simple English Wikipedia, snapshot 2023-11-01)
- **URL:** https://huggingface.co/datasets/wikimedia/wikipedia
- **Maintainer:** Wikimedia Foundation
- **Schema:** columns `id`, `url`, `title`, `text`

## License

Wikipedia text content is licensed under **CC-BY-SA 4.0** (Creative Commons Attribution-ShareAlike 4.0).
- Original license: https://creativecommons.org/licenses/by-sa/4.0/
- Terms: free to share + adapt for any purpose, including commercial, with attribution + same-license sharing for derivatives

## Attribution

When using or redistributing these files (or derivatives), credit:

> Wikipedia contributors. (2023). [Article title]. *Simple English Wikipedia*. Retrieved from [article URL]. Licensed under CC-BY-SA 4.0.

## BibTeX

```bibtex
@misc{wikimedia_wikipedia_2023,
  title        = {Wikipedia (Simple English snapshot 20231101)},
  author       = {{Wikimedia Foundation}},
  year         = {2023},
  publisher    = {Wikimedia Foundation, hosted on HuggingFace Hub},
  url          = {https://huggingface.co/datasets/wikimedia/wikipedia},
  license      = {CC-BY-SA-4.0}
}
```

## Re-generate

To replace these samples with different Wikipedia articles:

1. Edit `tooling/fetch_w7_datasets.py` → `EDUCATIONAL_TOPICS` list
2. Run `python3 tooling/fetch_w7_datasets.py --app studybot`
