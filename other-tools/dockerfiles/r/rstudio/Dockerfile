## Based on rocker/hadleyverse maintained by Carl Boettiger and Dirk Eddelbuettel
## Based on
FROM rocker/hadleyverse

MAINTAINER itamtao@gmail.com

## install the required R packages
RUN Rscript -e 'install.packages(pkgs = "qtl", dependencies = TRUE)'
RUN Rscript -e 'source("http://bioconductor.org/biocLite.R"); biocLite(c("DOQTL"), ask=FALSE)'
RUN Rscript -e 'devtools::install_github(repo = "dmgatti/DOQTL")'
RUN Rscript -e 'install.packages(c("yaml", "jsonlite", "data.table", "RcppEigen"), dependencies = TRUE)'
RUN Rscript -e 'devtools::install_github(repo = c("rqtl/qtl2geno", "rqtl/qtl2scan"))'
RUN Rscript -e 'devtools::install_github("simecek/intermediate")'
RUN Rscript -e 'devtools::install_github("kbroman/lmmlite")'
ADD assign-user-and-shared-folder.sh /etc/bootstrap.sh
