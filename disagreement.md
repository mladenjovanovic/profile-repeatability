
# Disagreement between two profiles

Disagreement estimation models

``` r
# --------------------------------
disagreement_mladen <- function(L1, V1, L2, V2) {
  ((abs(L1 - L2) * min(V1, V2)) + (abs(V1 - V2) * min(L1, L2)) + (abs(L1 - L2) * abs(V1 - V2))) / 2
}

# --------------------------------
disagreement_wu <- function(L1, V1, L2, V2) { # L1-to-V1, L2-to-V2

  triangle_area <- function(b, h) {
    0.5 * b * h
  }

  find_intersect <- function(x1, x2, y1, y2) {
    a <- y1 - y2
    b <- x1 * x2
    c <- x1 * y1 - x2 * y2

    (a * b) / c
  }

  if ((L2 >= L1 && V2 >= V1) || (L2 <= L1 && V2 <= V1)) {
    triangle_area(max(L1, L2), max(V1, V2)) - triangle_area(min(L1, L2), min(V1, V2))
  } else {
    # find intersect
    x <- find_intersect(L1, L2, V2, V1)
    z <- -(V1 / L1) * x + V1

    area_1 <- triangle_area(L2 - L1, z)

    area_2 <- triangle_area(V1 - V2, x)

    abs(area_1 + area_2)
  }
}

# ---------------------------------------
disagreement_euler <- function(L1, V1, L2, V2, step = 10^-5) {
  slope1 <- -V1 / L1
  slope2 <- -V2 / L2

  max_x <- max(L1, L2)
  x_i <- 0
  surface <- 0

  while (x_i <= max_x-step) {
    x_i_next <- x_i + step
    
    # First profile
    if (x_i < L1) {
      V1_i <- V1 + slope1 * x_i
    } else {
      V1_i <- 0
    }

    if (x_i_next < L1) {
      V1_i_next <- V1 + slope1 * x_i_next
    } else {
      V1_i_next <- 0
    }

    surface_1 <- step * V1_i_next + step * (V1_i - V1_i_next) / 2

    # Second profile
    if (x_i < L2) {
      V2_i <- V2 + slope2 * x_i
    } else {
      V2_i <- 0
    }

    if (x_i_next < L2) {
      V2_i_next <- V2 + slope2 * x_i_next
    } else {
      V2_i_next <- 0
    }

    surface_2 <- step * V2_i_next + step * (V2_i - V2_i_next) / 2

    surface <- surface + abs(surface_1 - surface_2)

    # Next iteration
    x_i <- x_i_next
  }


  return(surface)
}
```

Example of one profile being always better. The solution to this problem
is simple - it is the difference between the two triangle areas. But
let’s see how our estimation model works:

``` r
df <- tibble(
  L1 = 1,
  V1 = 1,
  ratio = seq(1, 2, length.out = 5),
  L2 = L1 * ratio,
  V2 = V1 * ratio
)
df$profile <- seq_along(df)

plot_profiles <- function(df) {
  plot_data <- pmap_dfr(df, function(...) {
    current <- tibble(...)
    x <- rbind(
      data.frame(
        ratio = current$ratio,
        profile = current$profile,
        A_L0 = c(0, current$L1),
        A_V0 = c(current$V1, 0),
        B_L0 = c(0, current$L2),
        B_V0 = c(current$V2, 0)
      )
    )
  })

  panel_a <- ggplot(
    plot_data,
    aes(group = profile)
  ) +
    theme_cowplot(8) +
    geom_line(aes(x = B_L0, y = B_V0), color = "grey") +
    geom_line(aes(x = A_L0, y = A_V0), color = "red") +
    scale_x_continuous(expand = c(0, 0), limits = c(0, NA)) +
    scale_y_continuous(expand = c(0, 0), limits = c(0, NA)) +
    xlab("Load (L)") +
    ylab("Velocity (V)")

  df <- df %>%
    rowwise() %>%
    mutate(
      mladen = disagreement_mladen(L1, V1, L2, V2),
      wu = disagreement_wu(L1, V1, L2, V2),
      euler = disagreement_euler(L1, V1, L2, V2)
    )

  plot_data <- gather(df, key = "method", value = "value", -(1:6))

  panel_b <- ggplot(plot_data, aes(x = L2, y = value, color = method)) +
    theme_cowplot(8) +
    geom_line(alpha = 0.8) +
    ylab(NULL) +
    xlab("Second profile L0")

  plot_grid(panel_a, panel_b, nrow = 1)
}

plot_profiles(df)
```

<img src="disagreement_files/figure-gfm/unnamed-chunk-3-1.png" width="90%" style="display: block; margin: auto;" />

Example of one profile having x higher

``` r
df <- tibble(
  L1 = 1,
  V1 = 1,
  ratio = seq(1, 2, length.out = 5),
  L2 = L1 * ratio,
  V2 = V1
)
df$profile <- seq_along(df)

plot_profiles(df)
```

<img src="disagreement_files/figure-gfm/unnamed-chunk-4-1.png" width="90%" style="display: block; margin: auto;" />

Example of one profile having proportionally different x and y, while
having same surface

``` r
df <- tibble(
  L1 = 1,
  V1 = 1,
  ratio = seq(1, 2, length.out = 5),
  L2 = L1 * ratio,
  V2 = V1 / ratio
)
df$profile <- seq_along(df)

plot_profiles(df)
```

<img src="disagreement_files/figure-gfm/unnamed-chunk-5-1.png" width="90%" style="display: block; margin: auto;" />
