# https://www.gnu.org/software/make/manual/html_node/

# Find all the -ascii.txt files we want to compile
# Note the single quotes around the * expressions. The shell will incorrectly expand these otherwise, but we want to send the * directly to the find command.
SRC_DIRS := .
BUILD_DIR := .
SRCS := $(shell find $(SRC_DIRS) -name '*-ascii.txt')
PDFs := $(SRCS:-ascii.txt=.pdf)

.PHONY: default
default: all

all: $(PDFs)

%.svg : %-ascii.txt
	aasvg < $< > $@

%.pdf : %.svg
	rsvg-convert -f pdf -o $@ $<

