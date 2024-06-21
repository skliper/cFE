# Create executable with built in tarfs

# In theory could just set up an executable with the extra steps to create the tarred object
# Setting up to easily run from prep, might work from install but would need to sort out directories

set(TARFS_DEV_DIR ${MISSION_BINARY_DIR}${CMAKE_INSTALL_PREFIX}/tarfs_dev)
set(DEV_DIR ${MISSION_BINARY_DIR}${CMAKE_INSTALL_PREFIX}/devboard)

file(MAKE_DIRECTORY ${TARFS_DEV_DIR})

# Create the nonvol tar file
execute_process(
  COMMAND tar cf ${TARFS_DEV_DIR}/tarfile nonvol
  WORKING_DIRECTORY ${DEV_DIR})

# Create the tar file object
execute_process(
  COMMAND ${CMAKE_LINKER} -r --noinhibit-exe -o tarfile.o -b binary tarfile
  WORKING_DIRECTORY ${TARFS_DEV_DIR})

# Force symbol inclusion as needed for modules
set(INCLUDE_SYMBOLS "-u strncat -u fgetc -u fputc -u fprintf -u fwrite")

# Convert since execute command needs list not string with spaces
string(REPLACE " " ";" TARFS_FLAGS_LIST ${RTEMS_SYS_SPECS_FLAGS} " " ${RTEMS_BSP_C_FLAGS} " " ${INCLUDE_SYMBOLS})

file(GLOB devboard_exe "${MISSION_BINARY_DIR}${CMAKE_INSTALL_PREFIX}/devboard/*.exe")
foreach(executable ${devboard_exe})
  message("Processing executable: ${executable}")
  get_filename_component(BASE_NAME ${executable} NAME_WE)

  #### LINKS
  #### Does not work to replace -Wl,--export-dynamic with -rdynamic
  #### Does need -lrtemscpu
#  set(TARFS_PRELINK ${CMAKE_C_COMPILER} -B/opt/rcc-1.3.1-gcc/sparc-gaisler-rtems5/sparc-rtems5/leon3/lib -qbsp=leon3 -qrtems -mcpu=leon3 -fno-common -u Init -o ${BASE_NAME}.prelink -Wl,--export-dynamic  -Wl,--whole-archive ${executable} tarfile.o -Wl,--no-whole-archive -lrtemscpu)

  #### BEST SO FAR
#  set(TARFS_PRELINK ${CMAKE_C_COMPILER} ${TARFS_FLAGS_LIST} -u Init -o ${BASE_NAME}.prelink -Wl,--export-dynamic  -Wl,--whole-archive ${executable} tarfile.o -Wl,--no-whole-archive -lrtemscpu)

  # rtemscpu adds ~8MB of debug info, need to strip even if cFS release build
  # -Wl,--export-dynamic doesn't seem to do anything (same exact size either way), needs 2 step link
  # -Wl,--whole-archive and -Wl,--no-whole-archive have no impact either
  # -u Init has no impact
  # TODO investigate adding: -ffunction-sections -fdata-sections -Wl,--gc-sections
  set(TARFS_LINK ${CMAKE_C_COMPILER} ${TARFS_FLAGS_LIST} ${executable} tarfile.o -lrtemscpu)

  #### LINKS
  #### First shot at rcc style prelink
#  set(TARFS_PRELINK ${CMAKE_C_COMPILER} -mcpu=leon3 -ffunction-sections -fdata-sections -Wl,--gc-sections -B/opt/rcc-1.3.1-gcc/sparc-gaisler-rtems5/leon3/lib -qrtems -specs bsp_specs  -o rki.elf.pre ${executable} tarfile.o --start-group -lrtemsbsp -lrtemscpu -latomic -lc -lm -lgcc --end-group)

  #### LINKS
#  set(TARFS_PRELINK ${CMAKE_C_COMPILER} -mcpu=leon3 -ffunction-sections -fdata-sections -Wl,--gc-sections -B/opt/rcc-1.3.1-gcc/sparc-gaisler-rtems5/leon3/lib -qrtems -o ${BASE_NAME}.prelink ${executable} tarfile.o -lrtemsbsp -lrtemscpu -latomic -lc -lm -lgcc)

  #### BROKEN - Note this fails since it includes crt0.c, and can't find _start ?!?
#  set(TARFS_PRELINK ${CMAKE_C_COMPILER} -mcpu=leon3 -ffunction-sections -fdata-sections -Wl,--gc-sections ${RTEMS_SYS_SPECS_FLAGS} -o ${BASE_NAME}.prelink ${executable} tarfile.o -latomic -lc -lm -lgcc)

  message("")
  message("Prelink execution:")
  execute_process(
    COMMAND ${TARFS_LINK} -o ${BASE_NAME}.prelink
    WORKING_DIRECTORY ${TARFS_DEV_DIR}
    COMMAND_ECHO STDOUT
    OUTPUT_VARIABLE TARFS_PRELINK_OUT
    ERROR_VARIABLE TARFS_PRELINK_ERR)
  message("Output:\n${TARFS_PRELINK_OUT}Error:\n${TARFS_PRELINK_ERR}")

  message("Get dl symbols")
  execute_process(
    COMMAND ${RTEMS_TOOLS_PREFIX}/bin/rtems-syms -v -e -c ${RTEMS_BSP_C_FLAGS} -C ${CMAKE_C_COMPILER} -o ${BASE_NAME}-dl-sym.o ${BASE_NAME}.prelink
    WORKING_DIRECTORY ${TARFS_DEV_DIR}
    COMMAND_ECHO STDOUT
    OUTPUT_VARIABLE TARFS_PRELINK_OUT
    ERROR_VARIABLE TARFS_PRELINK_ERR)
  message("Output:\n${TARFS_PRELINK_OUT}Error:\n${TARFS_PRELINK_ERR}")

  message("Final link:")
  execute_process(
    COMMAND ${TARFS_LINK} ${BASE_NAME}-dl-sym.o -o ${BASE_NAME}.elf
    WORKING_DIRECTORY ${TARFS_DEV_DIR}
    COMMAND_ECHO STDOUT
    OUTPUT_VARIABLE TARFS_PRELINK_OUT
    ERROR_VARIABLE TARFS_PRELINK_ERR)
  message("Output:\n${TARFS_PRELINK_OUT}Error:\n${TARFS_PRELINK_ERR}")

  message("Strip binary:")
  execute_process(
    COMMAND ${CMAKE_OBJCOPY} -O binary --strip-all ${BASE_NAME}.elf ${BASE_NAME}.bin
    WORKING_DIRECTORY ${TARFS_DEV_DIR}
    COMMAND_ECHO STDOUT
    OUTPUT_VARIABLE TARFS_PRELINK_OUT
    ERROR_VARIABLE TARFS_PRELINK_ERR)
  message("Output:\n${TARFS_PRELINK_OUT}Error:\n${TARFS_PRELINK_ERR}")

  message("Strip elf:")
  execute_process(
    COMMAND ${CMAKE_OBJCOPY} --strip-all ${BASE_NAME}.elf ${BASE_NAME}-strip.elf
    WORKING_DIRECTORY ${TARFS_DEV_DIR}
    COMMAND_ECHO STDOUT
    OUTPUT_VARIABLE TARFS_PRELINK_OUT
    ERROR_VARIABLE TARFS_PRELINK_ERR)
  message("Output:\n${TARFS_PRELINK_OUT}Error:\n${TARFS_PRELINK_ERR}")

  message("Report sizes:")
  execute_process(
    COMMAND ${CMAKE_SIZE} ${BASE_NAME}.elf
    WORKING_DIRECTORY ${TARFS_DEV_DIR}
    COMMAND_ECHO STDOUT
    OUTPUT_VARIABLE TARFS_PRELINK_OUT
    ERROR_VARIABLE TARFS_PRELINK_ERR)
  message("Output:\n${TARFS_PRELINK_OUT}Error:\n${TARFS_PRELINK_ERR}")

endforeach()
