## Preliminary steps
To begin, a genome sequence and a set of genomic loci must be defined. Let's 
focus on the C. elegans genome for now, and more specifically around its TSSs. 

```r
require(magrittr)
require(GenomicRanges)
require(ggplot2)
ce_seq <- Biostrings::getSeq(BSgenome.Celegans.UCSC.ce11::BSgenome.Celegans.UCSC.ce11)
ce_REs <- readRDS(url('http://ahringerlab.com/perioDNA/ce11_annotated_REs.rds'))
ce_proms <- ce_REs %>% 
    '['(.$is.prom) 
ce_TSSs <- ce_proms %>% 
    deconvolveBidirectionalPromoters() %>% 
    alignToTSS(50, 300) %>%
    withSeq(ce_seq)
list_TSSs <- list(
    "Ubiquitous TSSs" = ce_TSSs[ce_TSSs$which.tissues == 'Ubiq.'],
    "Germline TSSs" = ce_TSSs[ce_TSSs$which.tissues == 'Germline'], 
    "Neurons TSSs" = ce_TSSs[ce_TSSs$which.tissues == 'Neurons'], 
    "Muscle TSSs" = ce_TSSs[ce_TSSs$which.tissues == 'Muscle'], 
    "Hypodermis TSSs" = ce_TSSs[ce_TSSs$which.tissues == 'Hypod.'], 
    "Intestine TSSs" = ce_TSSs[ce_TSSs$which.tissues == 'Intest.']
)
```

## How perioDNA works

When running the `r getPeriodicity()` function for a given motif over a set of 
GRanges, R does the following steps: 

1. It first find all the possible motif occurences over the GRanges;  
2. It then find all the possible pairwise distances and generates a normalized 
histogram of the pairwise distances;  
3. It performs a Fast Fourier Transform of the normalized histogram; 
4. Finally, it normalizes the Power Spectrum Density (PSD) values (from FFT) to 
the average PSD to identify enriched periods in the input. 

```r
ubiq_TT <- getPeriodicity(
    ce_TSSs[ce_TSSs$which.tissues == 'Ubiq.'], 
    genome = ce_seq, 
    motif = 'TT', 
    freq = 0.10, 
    cores = 4
)
ggsave(
    plot = ggpubr::ggarrange(plotlist = plotPeriodicityResults(ubiq_TT), nrow = 1, ncol = 3),
    filename = paste0('examples/ubiquitous-promoters_TT-periodicity.pdf'), 
    width = 9, height = 3
)
``` 

![](../ubiquitous-promoters_TT-periodicity.png)

## Advanced use of getPeriodicity

We can split promoters by classes (e.g. tissue-specific promoters) and 
look at the periodicity of each set. 

```r    
MOTIF <- 'TT'
list_periodicities <- lapply(list_TSSs, function(proms) {
    getPeriodicity(
        proms, 
        genome = ce_seq,
        motif = MOTIF, 
        freq = 0.10, 
        cores = 40, 
        verbose = FALSE
    )
}) %>% setNames(names(list_TSSs))
dists <- lapply(list_periodicities, '[[', 'normalized_hist') %>% 
    namedListToLongFormat()
plots_dists <- ggplot(dists, aes(x = distance, y = norm_counts, color = name)) + 
    geom_line() +
    theme_bw() + 
    ylim(c(-1.5, 1.5)) +
    xlim(c(0, 200)) +
    labs(
        x = paste0('Distance between pairs of ', MOTIF), 
        y = 'Normalized counts', 
        title = paste0('Distribution of distances between pairs of ', MOTIF)
    ) + 
    facet_wrap(~name, nrow = 1) + 
    theme(legend.position = "none") + 
    scale_color_manual(
        values = c('#991919', '#1232D9', '#3B9B46', '#D99B12', '#9e9e9e', '#D912D4')
    )
PSDs <- lapply(list_periodicities, '[[', 'PSD') %>% 
    namedListToLongFormat() %>% 
    dplyr::mutate(Period = 1/freq) %>% 
    dplyr::filter(Period < 50)
plots_PSDs <- ggplot(PSDs, aes(x = Period, y = PSD, color = name)) + 
    geom_point() + 
    geom_line(stat = 'identity') +
    theme_bw() + 
    labs(
        x = 'Dinucleotide period', 
        y = 'Periodicity power spectrum density', 
        title = 'Power spectrum density of dinucleotide periodicity for different classes of promoters'
    ) + 
    facet_wrap(~name, nrow = 1) + 
    theme(legend.position = "none") + 
    scale_color_manual(
        values = c('#991919', '#1232D9', '#3B9B46', '#D99B12', '#9e9e9e', '#D912D4')
    )
p <- ggpubr::ggarrange(plotlist = list(plots_dists, plots_PSDs), nrow = 2, ncol = 1)
ggsave(paste0('examples/', MOTIF , '_tissue-specific-classes.pdf'), width = 15, height= 7)
```

![](../TT_tissue-specific-classes.png)

One can even perform dinucleotides periodicity analysis for multiple 
dinucleotides around several groups of promoters at once, as follows: 

```r
list_motifs <- c('WW', 'AA', 'TT', 'TA', 'AT')
list_periodicities <- lapply(list_motifs, function(MOTIF) {
    message('\t', MOTIF, ' periodicity...')
    lapply(list_TSSs, function(granges) {
        getPeriodicity(
            granges, 
            genome = ce_seq,
            motif = MOTIF,
            freq = 0.10, 
            cores = 120, 
            verbose = FALSE
        )
    })
}) %>% setNames(list_motifs)
dists <- lapply(list_periodicities, function(L) {
    lapply(L, '[[', 'normalized_hist') %>% namedListToLongFormat()
}) %>% namedListToLongFormat()
plots_dists <- ggplot(dists, aes(x = distance, y = norm_counts, color = name)) + 
    geom_line() +
    theme_bw() + 
    ylim(c(-1.5, 1.5)) +
    xlim(c(0, 200)) +
    labs(
        x = 'Distance between pairs of dinucleotides', 
        y = 'Normalized counts', 
        title = 'Distribution of distances between pairs of dinucleotides'
    ) + 
    facet_grid(name.1~name) + 
    theme(legend.position = "none") + 
    scale_color_manual(
        values = c('#991919', '#1232D9', '#3B9B46', '#D99B12', '#9e9e9e', '#D912D4')
    )
PSDs <- lapply(list_periodicities, function(L) {
    lapply(L, '[[', 'PSD') %>% namedListToLongFormat()
}) %>% namedListToLongFormat() %>% 
    dplyr::mutate(Period = 1/freq) %>% 
    dplyr::filter(Period < 50)
plots_PSDs <- ggplot(PSDs, aes(x = Period, y = PSD, color = name)) + 
    geom_point() + 
    geom_line(stat = 'identity') +
    theme_bw() + 
    ylim(c(0, 6)) + 
    labs(
        x = 'Dinucleotide period', 
        y = 'Periodicity power spectrum density', 
        title = 'Power spectrum density of dinucleotide periodicity for different classes of promoters'
    ) + 
    facet_grid(name.1~name) + 
    theme(legend.position = "none") + 
    scale_color_manual(
        values = c('#991919', '#1232D9', '#3B9B46', '#D99B12', '#9e9e9e', '#D912D4')
    )
p <- ggpubr::ggarrange(plotlist = list(plots_dists, plots_PSDs), nrow = 2, ncol = 1)
ggsave(
    paste0('examples/dinucleotides-PSDs_', paste0(list_motifs, collapse = '-'), '.pdf'), 
    width = 15, 
    height = 25
)
```

![PSDs_WW-AA-TT](../examples/png/dinucleotides-PSDs_WW-AA-TT-TA-AT.png)

## Plotting dinucleotide periodicity over GRanges

One can use the convenient plotting function to investigate the resulting track. 
For instance, we can look at the strength of periodicity over different classes 
of tissue-specific promoters. 

```r
periodicity_track <- rtracklayer::import.bw(
    'http://ahringerlab.com/perioDNA/TT-10-bp-periodicity.bw',
    as = 'Rle'
) %>% scaleBigWigs()
list_granges <- list(
    "Ubiquitous TSSs" = ce_proms[ce_proms$which.tissues == 'Ubiq.' & strand(ce_proms) == '+'] %>% 
        alignToTSS(500, 500),
    "Germline TSSs" = ce_proms[ce_proms$which.tissues == 'Germline' & strand(ce_proms) == '+'] %>% 
        alignToTSS(500, 500), 
    "Neurons TSSs" = ce_proms[ce_proms$which.tissues == 'Neurons' & strand(ce_proms) == '+'] %>% 
        alignToTSS(500, 500), 
    "Muscle TSSs" = ce_proms[ce_proms$which.tissues == 'Muscle' & strand(ce_proms) == '+'] %>% 
        alignToTSS(500, 500), 
    "Hypodermis TSSs" = ce_proms[ce_proms$which.tissues == 'Hypod.' & strand(ce_proms) == '+'] %>% 
        alignToTSS(500, 500), 
    "Intestine TSSs" = ce_proms[ce_proms$which.tissues == 'Intest.' & strand(ce_proms) == '+'] %>% 
        alignToTSS(500, 500)
)
p <- plotAggregateCoverage(
    periodicity_track, 
    list_granges, 
    ylab = '10-bp periodicity strength', 
    xlab = 'Distance from TSS'
)
ggsave(
    filename = 'examples/TT-10bp-periodicity_tissue-spe-TSSs.pdf', 
    height = 5, 
    width = 7
)
```

![TT](../examples/png/TT-10bp-periodicity_tissue-spe-TSSs.png)

Or using multiple periodicity tracks: 

```r
list_periodicity_tracks <- list(
    'WW' = rtracklayer::import.bw(
        'http://ahringerlab.com/perioDNA/WW-10-bp-periodicity.bw',
        as = 'Rle'
    ) %>% scaleBigWigs(),
    'TT' = rtracklayer::import.bw(
        'http://ahringerlab.com/perioDNA/TT-10-bp-periodicity.bw',
        as = 'Rle'
    ) %>% scaleBigWigs(),
    'AA' = rtracklayer::import.bw(
        'http://ahringerlab.com/perioDNA/AA-10-bp-periodicity.bw',
        as = 'Rle'
    ) %>% scaleBigWigs()
)
p <- plotAggregateCoverage(
    list_periodicity_tracks, 
    list_granges, 
    ylab = '10-bp periodicity strength', 
    xlab = 'Distance from TSS', 
    split_by_granges = TRUE
)
ggsave(
    filename = 'examples/dinucleotides-10bp-periodicity_nuc-occ_tissue-spe-TSSs.pdf', 
    height = 3, 
    width = 3 * length(list_granges)
)
```

![multiDiNuc](../examples/png/dinucleotides-10bp-periodicity_nuc-occ_tissue-spe-TSSs.png)

This clearly highlights the increase of WW 10-bp periodicity immediately 
downstream of ubiquitous and germline promoters, while it is 
inexistent downstream of other TSSs.  
We can also notice that this is primarily due to the underlying TT periodicity.