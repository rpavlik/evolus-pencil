cmake_minimum_required(VERSION 2.8)
project(Pencil NONE)

set(MKDIR "${CMAKE_COMMAND}" -E make_directory)
set(COPY_IF_DIFFERENT "${CMAKE_COMMAND}" -E copy_if_different)

set(STAGEDIR "${CMAKE_CURRENT_BINARY_DIR}/stage")

include("${CMAKE_CURRENT_SOURCE_DIR}/files.cmake")
include("${CMAKE_CURRENT_SOURCE_DIR}/version.cmake")


function(add_configure_command STAGE INFILE OUTFILE)
	get_filename_component(outdir "${OUTFILE}" PATH)
	file(RELATIVE_PATH relsrc "${CMAKE_CURRENT_SOURCE_DIR}/app" "${INFILE}")
	get_filename_component(outdir "${OUTFILE}" PATH)
	add_custom_command(OUTPUT "${OUTFILE}"
		COMMAND ${MKDIR} "${outdir}"
		COMMAND "${CMAKE_COMMAND}" -DINFILE:FILEPATH="${INFILE}" -DOUTFILE:FILEPATH="${OUTFILE}" -P "${CMAKE_CURRENT_SOURCE_DIR}/configurefile.cmake"
		MAIN_DEPENDENCY "${INFILE}"
		WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
		DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/version.cmake" "${CMAKE_CURRENT_SOURCE_DIR}/configurefile.cmake"
		COMMENT "Configuring ${relsrc} into ${STAGE} staging directory...")
endfunction()

function(stage_files STAGE VARNAME)
	set(OUTFILES)
	set(STAGEDIR "${CMAKE_CURRENT_BINARY_DIR}/${STAGE}")
	foreach(fn ${FILES})
		set(outfn "${STAGEDIR}/${fn}")
		get_filename_component(outdir "${outfn}" PATH)
		set(infn "${CMAKE_CURRENT_SOURCE_DIR}/app/${fn}")
	
		list(APPEND OUTFILES "${outfn}")
		add_custom_command(OUTPUT "${outfn}"
			COMMAND ${MKDIR} "${outdir}"
			COMMAND ${COPY_IF_DIFFERENT} "${infn}" "${outfn}"
			MAIN_DEPENDENCY "${infn}"
			COMMENT "Copying ${fn} to ${STAGE} staging directory...")
	endforeach()
	set(CONFIGURE_FILES
		content/pencil/mainWindow.xul
		content/pencil/aboutDialog.xul
		content/pencil/common/pencil.js)
	foreach(fn ${CONFIGURE_FILES})
		set(outfn "${STAGEDIR}/${fn}")
		get_filename_component(outdir "${outfn}" PATH)
		set(infn "${CMAKE_CURRENT_SOURCE_DIR}/app/${fn}")
	
		list(APPEND OUTFILES "${outfn}")
		add_configure_command(${STAGE} "${infn}" "${outfn}")
	endforeach()
	
	set(${VARNAME} ${OUTFILES} PARENT_SCOPE)
endfunction()

stage_files(XPI XPIOUTFILES)
add_configure_command(XPI "${CMAKE_CURRENT_SOURCE_DIR}/app/install.rdf.tpl.xml" "${CMAKE_CURRENT_BINARY_DIR}/XPI/install.rdf")
add_configure_command(XPI "${CMAKE_CURRENT_SOURCE_DIR}/app/update.rdf.tpl.xml" "${CMAKE_CURRENT_BINARY_DIR}/XPI/update.rdf")
list(APPEND XPIOUTFILES "${CMAKE_CURRENT_BINARY_DIR}/XPI/install.rdf" "${CMAKE_CURRENT_BINARY_DIR}/XPI/update.rdf")

add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/${XPI_NAME}"
	COMMAND zip -r "${CMAKE_CURRENT_BINARY_DIR}/${XPI_NAME}" *
	WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/XPI/"
	DEPENDS ${XPIOUTFILES}
	COMMENT "Creating ${XPI_NAME}")

add_custom_target(xpi ALL
	DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/${XPI_NAME}"
	COMMENT "Built XPI file: ${CMAKE_CURRENT_BINARY_DIR}/${XPI_NAME}")
