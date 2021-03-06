cmake_minimum_required ( VERSION 3.3 )

FOREACH (policy CMP0110)
	IF (POLICY ${policy})
		CMAKE_POLICY(SET ${policy} NEW)
	ENDIF ()
ENDFOREACH ()

# will add the tests from test suite
# each test will be added as separate one

function (test_prop test)
	SET_TESTS_PROPERTIES ( "${test}" PROPERTIES SKIP_RETURN_CODE 42 )
	SET_TESTS_PROPERTIES ( "${test}" PROPERTIES FIXTURES_REQUIRED UBER )
	SET_TESTS_PROPERTIES ( "${test}" PROPERTIES RESOURCE_LOCK DbAccess )
endfunction()

if ( DISABLE_TESTING )
	return()
endif()

	
find_program(CYGPATH NAMES cygpath)
diag(CYGPATH)
if (CYGPATH AND WIN32)
	execute_process ( COMMAND "${CYGPATH}" -u "${MANTICORE_BINARY_DIR}"
					OUTPUT_VARIABLE MANTICORE_SCRIPTING_DIR
					ERROR_QUIET
					OUTPUT_STRIP_TRAILING_WHITESPACE )
					
else()
	set(MANTICORE_SCRIPTING_DIR "${MANTICORE_BINARY_DIR}")
endif()

mark_as_advanced (CYGPATH)

diag(CMAKE_CURRENT_SOURCE_DIR)
diag(MANTICORE_BINARY_DIR)
diag(MANTICORE_SCRIPTING_DIR)

file ( GLOB tests "test_[0-9]*" )
list ( SORT tests )

if ( NOT "${CMAKE_CURRENT_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_BINARY_DIR}")
	message ( STATUS "Create ubertest.php proxy" )
	configure_file ( "ubertest.php.in" "${MANTICORE_BINARY_DIR}/test/ubertest.php" @ONLY )
endif()
	
if ( NOT SEARCHD_CLI_EXTRA )
	set ( SEARCHD_CLI_EXTRA "" )
endif()

# since cmake 3.18 names need fixup
include(fixup_test_name)

fixup_test_name(tst "Cleaning guess cache")
add_test ( NAME ${tst}
		#    	CONFIGURATIONS Debug
		COMMAND ${CMAKE_COMMAND} -E remove guess.txt )
SET_TESTS_PROPERTIES (${tst} PROPERTIES FIXTURES_SETUP UBER )

fixup_test_name(tst "Cleaning errors")
add_test ( NAME ${tst}
		COMMAND ${CMAKE_COMMAND} -E remove error_all.txt )
SET_TESTS_PROPERTIES ( ${tst} PROPERTIES FIXTURES_SETUP UBER )

foreach ( test ${tests} )
	# get the last dir (filename)
	get_filename_component ( onetest "${test}" NAME_WE )
	# crop the 'test_' heading (so, only number is rest)
	STRING ( REGEX REPLACE "^test_" "" onetest "${onetest}" )

	set (REQUIRES "")
	# check if we can extract the name for the test from it
	if ( EXISTS "${test}/test.xml" )
		# open the file with the test and read _chunk of 512 bytes
		file ( READ ${test}/test.xml _test_head LIMIT 512 )

		# convert it to lower (CMAKE regexes are case-sensitive)
		STRING ( TOLOWER "${_test_head}" _lower_test_head )

		# extract test name. It is not simple 'match <name>(.*)</name>' since we want non-lower-case,
		# so find the chunk in lowered-case, then take the clause by substring from original chunk

		# find pos and length of test name
		STRING ( REGEX MATCH "(^.*<name>)(.*)</name>" _chunk "${_lower_test_head}" )
		STRING ( LENGTH "${CMAKE_MATCH_1}" _chunk )
		STRING ( LENGTH "${CMAKE_MATCH_2}" _length )
		# cut (substring) test name from original chunk
		STRING ( SUBSTRING "${_test_head}" ${_chunk} ${_length} test_name )

		# extract test requirements
		IF ( "${_test_head}" MATCHES "<requires>(.*)</requires>" )
			set ( _requires "${CMAKE_MATCH_1}" )
			STRING ( REGEX REPLACE "<!--.*-->" "" _requires "${_requires}" ) # remove comments
			STRING ( REGEX REPLACE "[ \t\n]" "" _requires "${_requires}" ) # remove comments
			STRING ( REGEX REPLACE "/><" ";" _requires "${_requires}" ) # change gaps to ;
			STRING ( REGEX REPLACE "/>" "" _requires "${_requires}" ) # remove />
			STRING ( REGEX REPLACE "<" "" _requires "${_requires}" ) # remove <

			foreach ( require ${_requires} )
				string ( STRIP ${require} require )
				STRING ( TOUPPER "${require}" require )
				LIST (APPEND REQUIRES ${require})
			endforeach ()

		endif ()

	else ()
		if ( onetest STREQUAL "028" ) # the only non-xml test we have
			set ( test_name "spelldump...")
		else()
			set ( test_name "" )
		endif()
	endif ()

	set ( ADD_TEST 1 )
	if ( WIN32 OR CYGWIN )
		if ("NON-WINDOWS" IN_LIST REQUIRES )
			set ( ADD_TEST 0)
			set ( EXPLANATION "non-windows")
		endif()
	else()
		if ( "WINDOWS" IN_LIST REQUIRES )
			set ( ADD_TEST 0)
			set ( EXPLANATION "windows" )
		endif()
	endif()

	if ( NOT USE_ODBC AND "ODBC" IN_LIST REQUIRES )
		set ( ADD_TEST 0 )
		set ( EXPLANATION "ODBC provider" )
	endif ()

	if ( NOT USE_RE2 AND "RE2" IN_LIST REQUIRES )
		set ( ADD_TEST 0)
		set ( EXPLANATION "RE2" )
	endif()

	if ( NOT USE_ICU AND "ICU" IN_LIST REQUIRES )
		set ( ADD_TEST 0 )
		set ( EXPLANATION "ICU" )
	endif ()

	if ( NOT USE_GALERA AND "REPLICATION" IN_LIST REQUIRES )
		set ( ADD_TEST 0 )
		set ( EXPLANATION "Galera library for replication" )
	endif ()

	STRING(CONFIGURE "${onetest} -- ${test_name}" tst_name @ONLY ESCAPE_QUOTES)
	if ( ADD_TEST )
		# conditionally add plain test
		if ( NOT "FORCE-RT" IN_LIST REQUIRES )
			fixup_test_name (tstn "test_${tst_name}")
			add_test ( NAME "${tstn}"
					#    	CONFIGURATIONS Debug
					WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
					COMMAND php ubertest.php
						-b "${MANTICORE_BINARY_DIR}/src/"
						-t "${MANTICORE_BINARY_DIR}/test/"
						-tt "${MANTICORE_SCRIPTING_DIR}/test/"
						--ctest -u test --strict-verbose --no-demo
						"--sd_extra" "${SEARCHD_CLI_EXTRA}"
						t ${onetest} )

			SET (labels "PLAIN")
			foreach ( require ${REQUIRES} )
				LIST (APPEND labels ${require})
			endforeach ()
			SET_TESTS_PROPERTIES ( "${tstn}" PROPERTIES LABELS "${labels}" )
			if ("HEAVY" IN_LIST REQUIRES)
				SET_TESTS_PROPERTIES("${tstn}" PROPERTIES TIMEOUT "1800")
			endif ()
			test_prop ( "${tstn}" )
		endif()

		# conditionally add rt test
		if ( NOT "NON-RT" IN_LIST REQUIRES )
			fixup_test_name(tstn "rt_${tst_name}")
			add_test ( NAME "${tstn}"
					#			CONFIGURATIONS "Debug"
					WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
					COMMAND php ubertest.php
					-b "${MANTICORE_BINARY_DIR}/src/"
					-t "${MANTICORE_BINARY_DIR}/test/"
					-tt "${MANTICORE_SCRIPTING_DIR}/test/"
					--ctest -u test --strict-verbose --no-demo
					"--sd_extra" "${SEARCHD_CLI_EXTRA}"
					--rt --ignore-weights
					t ${onetest} )
			SET ( labels "RT" )
			foreach ( require ${REQUIRES} )
				LIST ( APPEND labels ${require} )
			endforeach ()
			SET_TESTS_PROPERTIES ( "${tstn}" PROPERTIES LABELS "${labels}" )
			if ("HEAVY" IN_LIST REQUIRES)
				SET_TESTS_PROPERTIES("${tstn}" PROPERTIES TIMEOUT "1800")
			endif ()
			test_prop ( "${tstn}" )
		endif()
	else()
		message (STATUS "Skip adding test ${tst_name} since it requires ${EXPLANATION}")
	endif(ADD_TEST)
#		SET_TESTS_PROPERTIES ( "\"test_${ts)t_name}\"" "\"rt_${tst_name}\"" PROPERTIES ATTACHED_FILES_ON_FAIL "${MANTICORE_BINARY_DIR}/test/test_${onetest}/report.txt" )

endforeach ()




