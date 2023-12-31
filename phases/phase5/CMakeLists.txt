cmake_minimum_required(VERSION 3.12)
project(wasmlinux-phase5 C)

set(host ${CMAKE_CURRENT_LIST_DIR}/../../host)

set(kmdist)
set(umdist)
if(KMDIST)
  set(kmdist ${KMDIST})
endif()
if(UMDIST)
  set(umdist ${UMDIST})
endif()

if(NOT KMDIST AND EXISTS ${CMAKE_CURRENT_LIST_DIR}/../../kmdist)
  set(kmdist ${CMAKE_CURRENT_LIST_DIR}/../../kmdist)
endif()

if(NOT UMDIST AND EXISTS ${CMAKE_CURRENT_LIST_DIR}/../../umdist)
  set(umdist ${CMAKE_CURRENT_LIST_DIR}/../../umdist)
endif()

set(wasmmodules)
set(userhashes)

function(hashpath var hash)
  if("${hash}" MATCHES "(..)(..)(.*)")
    set(${var} ${CMAKE_MATCH_1}/${CMAKE_MATCH_2}/${CMAKE_MATCH_3}
      PARENT_SCOPE)
  else()
    message(FATAL_ERROR "Invalid hashpath: ${hash}")
  endif()
endfunction()

function(runwasm2c wasm modname)
  get_filename_component(dir ${wasm} DIRECTORY)
  add_custom_command(OUTPUT 
    ${dir}/${modname}.c
    ${dir}/${modname}.h
    COMMAND ${host}/bin/wasm2c
    --module-name=${modname}
    --enable-threads
    -o ${modname}.c
    ${wasm}
    COMMAND ${CMAKE_COMMAND}
    -Dhdr=${dir}/${modname}.h
    -P ${CMAKE_CURRENT_LIST_DIR}/w2cfixup.cmake
    DEPENDS ${wasm} ${host}/bin/wasm2c
    ${CMAKE_CURRENT_LIST_DIR}/w2cfixup.cmake
    WORKING_DIRECTORY ${dir}
    COMMENT "wasm2c ${modname}"
    )
  add_custom_target(${modname}-wasm2c
    ALL
    DEPENDS ${dir}/${modname}.c
    ${dir}/${modname}.h)
endfunction()

if(kmdist)
  runwasm2c(${kmdist}/lin.wasm kernel)
endif()

if(umdist)
  # Enumerate userland modules
  file(GLOB binfiles LIST_DIRECTORIES false
    CONFIGURE_DEPENDS ${umdist}/bin/*)
  foreach(e ${binfiles})
    if(IS_SYMLINK "${e}" OR IS_DIRECTORY "${e}")
      # Skip
    else()
      file(READ ${e} hex OFFSET 0 LIMIT 4 HEX)
      file(READ ${e} maybeshebang OFFSET 0 LIMIT 2 HEX)
      if("${hex}" STREQUAL "0061736d") # \0asm (wasm module hdr)
        message(STATUS "${e}: Wasm module")
        list(APPEND wasmmodules ${e})
      elseif("${maybeshebang}" STREQUAL "2321") # #! (shebang)
        message(STATUS "${e}: shebang")
        file(STRINGS ${e} shebang NO_HEX_CONVERSION)
        list(LENGTH shebang len)
        if(${len} EQUAL 2)
          list(GET shebang 0 hdr)
          list(GET shebang 1 hash)
          if("${hdr}" STREQUAL "#!/usr/bin/env wasmlinux-id-exec")
            list(APPEND userhashes ${hash})
            get_filename_component(nam ${e} NAME)
            set(hash_${hash} ${nam})
            message(STATUS "... ${hash}")
          endif()
        endif()
      else()
        message(STATUS "${e}: UNKNOWN")
      endif()
    endif()
  endforeach()

  # Move raw wasmmodules and assign id
  foreach(e ${wasmmodules})
    file(SHA256 "${e}" hash)
    list(APPEND userhashes ${hash})
    hashpath(hp ${hash})
    set(destdir ${umdist}/wasmlinux/modcache/${hp})
    file(MAKE_DIRECTORY ${destdir})
    file(RENAME ${e} ${destdir}/base.wasm)
    message(STATUS "Cache: ${e} => ${destdir}")
    file(WRITE ${e} "#!/usr/bin/env wasmlinux-id-exec\n${hash}\n")
    get_filename_component(nam ${e} NAME)
    set(hash_${hash} ${nam})
  endforeach()

  # Remove Redundant modcache
  file(GLOB_RECURSE cc LIST_DIRECTORIES false
    RELATIVE ${umdist}/wasmlinux/modcache
    ${umdist}/wasmlinux/modcache/*.wasm)
  foreach(p ${cc})
    if("${p}" MATCHES "(..)/(..)/([^/]*)/base.wasm")
      set(hash ${CMAKE_MATCH_1}${CMAKE_MATCH_2}${CMAKE_MATCH_3})
      if(DEFINED hash_${hash})
        message(STATUS "Keep: ${p}")
      else()
        message(STATUS "Remove: ${p}")
        hashpath(hp ${hash})
        file(REMOVE_RECURSE ${umdist}/wasmlinux/modcache/${hp})
      endif()
    else()
      message(STATUS "Ignore: ${p}")
    endif()
  endforeach()

  # Assign unique name
  set(usernames)
  foreach(h ${userhashes})
    set(name ${hash_${h}})
    if(NOT DEFINED nam_${name})
      set(nam_${name} ${h})
      list(APPEND usernames ${name})
    else()
      message(FATAL_ERROR "DUPE!: ${nam_${hash_${h}}}")
    endif()
  endforeach()

  # Write modmap.tsv, invoke wasm2c 
  file(WRITE ${umdist}/wasmlinux/modcache/modmap.tsv "")
  foreach(n ${usernames})
    set(hash "${nam_${n}}")
    hashpath(hp ${hash})
    file(APPEND ${umdist}/wasmlinux/modcache/modmap.tsv "${hash}\t${n}")
    runwasm2c(${umdist}/wasmlinux/modcache/${hp}/base.wasm ${n}_base_w2c)
  endforeach()

endif()
