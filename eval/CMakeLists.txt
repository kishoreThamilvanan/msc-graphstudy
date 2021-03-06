#! /usr/bin/cmake -P
#! -*- coding:utf-8; mode:cmake; -*-

# Copyright (C) 2018 Karlsruhe Institute of Technology
# Copyright (C) 2018 Moritz Klammler <moritz.klammler@alumni.kit.edu>
#
# This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public
# License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later
# version.
#
# This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied
# warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License for more
# details.
#
# You should have received a copy of the GNU General Public License along with this program.  If not, see
# <http://www.gnu.org/licenses/>.

## The `do-*` targets are to be considered an implementation detail.  They shall not be built explicitly by the user.

set(min_xval_runs 5)
set(good_xval_runs 10)

set(xval_verbosity "--quiet")

set(all_properties rdf-global rdf-local angular edge-length princomp1st princomp2nd tension)
list(LENGTH all_properties property_count)
math(EXPR property_count_minus_one "${property_count} - 1")

set(puncpropfile "${CMAKE_CURRENT_BINARY_DIR}/puncture-properties.txt")
string(TIMESTAMP timestamp "%a, %d %b %Y %H:%M:%S")
file(WRITE  "${puncpropfile}" "#! /bin/false\n")
file(APPEND "${puncpropfile}" "#! -*- coding:utf-8; mode:conf-space; -*-\n\n")
file(APPEND "${puncpropfile}" "## THIS IS A GENERATED FILE; PLEASE DO NOT EDIT IT MANUALLY!\n")
file(APPEND "${puncpropfile}" "## Generated by CMake on ${timestamp}\n\n")
file(APPEND "${puncpropfile}" "## The following list contains ${property_count} properties\n\n")
foreach(prop ${all_properties})
    file(APPEND "${puncpropfile}" "${prop}\n")
endforeach(prop)

math(EXPR total_runs_punct1 "1 * ${property_count} * ${min_xval_runs}")
math(EXPR total_runs_punct2 "2 * ${property_count} * ${min_xval_runs}")

set(individual_xval_json_files
    xeval-01.json xeval-02.json xeval-03.json xeval-04.json xeval-05.json
    xeval-06.json xeval-07.json xeval-08.json xeval-09.json xeval-10.json
)

set(progressfile "${CMAKE_CURRENT_BINARY_DIR}/.progress")
set(progressscript "${CMAKE_CURRENT_SOURCE_DIR}/progress.py")

add_custom_target(
    eval-clean
    COMMAND "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_SOURCE_DIR}/cleanup.cmake"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    VERBATIM
    DEPENDS cleanup.cmake
)

add_custom_target(
    eval
    COMMAND "${Python3_EXECUTABLE}" "${progressscript}" "--file=${progressfile}"
            "--total=${total_runs_punct2}" "--total=${good_xval_runs}"
    COMMAND "${CMAKE_COMMAND}" --build "${PROJECT_BINARY_DIR}" --target do-puncture
    COMMAND "${CMAKE_COMMAND}" --build "${PROJECT_BINARY_DIR}" --target do-cross-valid
    COMMAND "${Python3_EXECUTABLE}" "${progressscript}" "--file=${progressfile}" "Evaluation complete!"
    COMMAND "${CMAKE_COMMAND}" -E remove -f "${progressfile}"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    VERBATIM
    DEPENDS omnibus eval-clean progress.py
)

set(partial_wrappers    cross-valid             puncture                puncture-excl           puncture-incl          )
set(partial_totalrun    ${good_xval_runs}       ${total_runs_punct2}    ${total_runs_punct1}    ${total_runs_punct1}   )
foreach(i RANGE 3)
    list(GET partial_wrappers ${i} what)
    list(GET partial_totalrun ${i} runs)
    add_custom_target(
        eval-${what}
        COMMAND "${Python3_EXECUTABLE}" "${progressscript}" "--file=${progressfile}" "--total=${runs}"
        COMMAND "${CMAKE_COMMAND}" --build "${PROJECT_BINARY_DIR}" --target do-${what}
        COMMAND "${Python3_EXECUTABLE}" "${progressscript}" "--file=${progressfile}" "Partial evaluation complete!"
        COMMAND "${CMAKE_COMMAND}" -E remove -f "${progressfile}"
        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
        VERBATIM
        DEPENDS omnibus eval-clean progress.py
    )
endforeach(i)

add_custom_target(do-puncture DEPENDS do-puncture-excl do-puncture-incl)
add_custom_target(do-puncture-excl)
add_custom_target(do-puncture-incl)

add_custom_target(
    do-cross-valid
    COMMAND "${Python3_EXECUTABLE}" "${progressscript}" "--file=${progressfile}"
            "--update=${good_xval_runs}" "Full Cross Validation"
    COMMAND "${Python3_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/cross-validate.py"
            "--runs=${good_xval_runs}"
            "--output=${CMAKE_CURRENT_BINARY_DIR}/cross-valid.json"
            "--directory=${CMAKE_CURRENT_BINARY_DIR}"
            "--huang=${CMAKE_CURRENT_BINARY_DIR}/huang-weights.json"
            "--datadir=${MSC_DATA_DIR}"
            "--puncture=0"
            -- "${Python3_EXECUTABLE}" -m driver.deploy ${xval_verbosity} --model
            "--configdir=${MSC_CONFIG_DIR}" "--datadir=${MSC_DATA_DIR}" "--bindir=${PROJECT_BINARY_DIR}"
    COMMAND "${CMAKE_COMMAND}" --build "${PROJECT_BINARY_DIR}" --target do-stamp-cross-valid
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
    VERBATIM
    BYPRODUCTS cross-valid.json huang-weights.json ${individual_xval_json_files}
    DEPENDS cross-validate.py progress.py
)

foreach(prop ${all_properties})
    add_custom_target(
        do-puncture-excl-${prop}
        COMMAND "${Python3_EXECUTABLE}" "${progressscript}" "--file=${progressfile}"
                "--update=${min_xval_runs}" "Punctured Cross Validation Excluding Only %(${prop})"
        COMMAND "${CMAKE_COMMAND}" -E copy_directory "${MSC_CONFIG_DIR}"
                "${CMAKE_CURRENT_BINARY_DIR}/config-without-${prop}/"
        COMMAND "${Python3_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/make-config-puncture.py"
                -x ${prop} -o "${CMAKE_CURRENT_BINARY_DIR}/config-without-${prop}/puncture.cfg"
        COMMAND "${Python3_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/cross-validate.py"
                "--runs=${min_xval_runs}"
                "--output=${CMAKE_CURRENT_BINARY_DIR}/cross-valid-without-${prop}.json"
                "--test=nn-forward"
                "--puncture=1"
                -- "${Python3_EXECUTABLE}" -m driver.deploy ${xval_verbosity} --model
                "--configdir=${CMAKE_CURRENT_BINARY_DIR}/config-without-${prop}/"
                "--datadir=${MSC_DATA_DIR}" "--bindir=${PROJECT_BINARY_DIR}"
        # Make sure we don't leave a misconfigured state behind
        COMMAND "${CMAKE_COMMAND}" -E env MSC_PUNCTURE=1
                "${Python3_EXECUTABLE}" -m driver.deploy ${xval_verbosity} --model --clean
                "--configdir=${CMAKE_CURRENT_BINARY_DIR}/config-without-${prop}/"
                "--datadir=${MSC_DATA_DIR}" "--bindir=${PROJECT_BINARY_DIR}"
        COMMAND "${CMAKE_COMMAND}" -E remove_directory "${CMAKE_CURRENT_BINARY_DIR}/config-without-${prop}/"
        COMMAND "${CMAKE_COMMAND}" --build "${PROJECT_BINARY_DIR}" --target do-stamp-puncture
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
        VERBATIM
        BYPRODUCTS cross-valid-without-${prop}.json
        DEPENDS make-config-puncture.py cross-validate.py progress.py
    )
    add_dependencies(do-puncture-excl do-puncture-excl-${prop})
endforeach(prop)

foreach(prop ${all_properties})
    add_custom_target(
        do-puncture-incl-${prop}
        COMMAND "${Python3_EXECUTABLE}" "${progressscript}" "--file=${progressfile}"
                "--update=${min_xval_runs}" "Punctured Cross Validation Including Only %(${prop})"
        COMMAND "${CMAKE_COMMAND}" -E copy_directory "${MSC_CONFIG_DIR}"
                "${CMAKE_CURRENT_BINARY_DIR}/config-with-${prop}/"
        COMMAND "${Python3_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/make-config-puncture.py"
                -y ${prop} -o "${CMAKE_CURRENT_BINARY_DIR}/config-with-${prop}/puncture.cfg"
        COMMAND "${Python3_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/cross-validate.py"
                "--runs=${min_xval_runs}"
                "--output=${CMAKE_CURRENT_BINARY_DIR}/cross-valid-with-${prop}.json"
                "--test=nn-forward"
                "--puncture=${property_count_minus_one}"
                -- "${Python3_EXECUTABLE}" -m driver.deploy ${xval_verbosity} --model
                "--configdir=${CMAKE_CURRENT_BINARY_DIR}/config-with-${prop}/"
                "--datadir=${MSC_DATA_DIR}" "--bindir=${PROJECT_BINARY_DIR}"
        # Make sure we don't leave a misconfigured state behind
        COMMAND "${CMAKE_COMMAND}" -E env MSC_PUNCTURE=${property_count_minus_one}
                "${Python3_EXECUTABLE}" -m driver.deploy ${xval_verbosity} --model --clean
                "--configdir=${CMAKE_CURRENT_BINARY_DIR}/config-with-${prop}/"
                "--datadir=${MSC_DATA_DIR}" "--bindir=${PROJECT_BINARY_DIR}"
        COMMAND "${CMAKE_COMMAND}" -E remove_directory "${CMAKE_CURRENT_BINARY_DIR}/config-with-${prop}/"
        COMMAND "${CMAKE_COMMAND}" --build "${PROJECT_BINARY_DIR}" --target do-stamp-puncture
        WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
        VERBATIM
        BYPRODUCTS cross-valid-with-${prop}.json
        DEPENDS make-config-puncture.py cross-validate.py progress.py
    )
    add_dependencies(do-puncture-incl do-puncture-incl-${prop})
endforeach(prop)

add_custom_target(
    do-stamp-cross-valid
    COMMAND "${CMAKE_COMMAND}" -E touch report/eval-cross-valid.stamp
    COMMAND "${CMAKE_COMMAND}" -E touch paper/eval-cross-valid.stamp
    COMMAND "${CMAKE_COMMAND}" -E touch slides_2018-04-24_kit/eval-cross-valid.stamp
    COMMAND "${CMAKE_COMMAND}" -E touch slides_2018-09-26_gd18/eval-cross-valid.stamp
    WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
)

add_custom_target(
    do-stamp-puncture
    COMMAND "${CMAKE_COMMAND}" -E touch report/eval-puncture.stamp
    COMMAND "${CMAKE_COMMAND}" -E touch paper/eval-puncture.stamp
    COMMAND "${CMAKE_COMMAND}" -E touch slides_2018-04-24_kit/eval-puncture.stamp
    COMMAND "${CMAKE_COMMAND}" -E touch slides_2018-09-26_gd18/eval-puncture.stamp
    WORKING_DIRECTORY "${PROJECT_BINARY_DIR}"
)
