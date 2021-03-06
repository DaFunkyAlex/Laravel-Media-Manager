<script>
import Utilities from './mixins/utils'
import Form from './mixins/form'
import FileFiltration from './mixins/filtration'
import BulkSelect from './mixins/bulk'
import LockFile from './mixins/lock'
import SelectedFile from './mixins/selected'
import Restriction from './mixins/restriction'
import Watchers from './mixins/watch'
import Computed from './mixins/computed'

import Bounty from 'vue-bounty'

export default {
    components: {Bounty},
    name: 'media-manager',
    mixins: [
        Utilities,
        Form,
        FileFiltration,
        BulkSelect,
        LockFile,
        SelectedFile,
        Restriction,
        Computed,
        Watchers
    ],
    props: [
        'baseUrl',
        'inModal',
        'hideFilesExt',
        'mediaTrans',
        'filesRoute',
        'dirsRoute',
        'lockFileRoute',
        'restrictPath',
        'uploadPanelImgList',
        'hideExt',
        'hidePath'
    ],
    data() {
        return {
            isLoading: false,
            no_files: false,
            loading_files: false,
            no_search: false,
            ajax_error: false,
            toggleInfo: true,
            toggleUploadArea: false,
            showProgress: false,
            progressCounter: 0,

            linkCopied: false,
            bulkSelectAll: false,
            bulkSelect: false,
            folderWarning: false,
            checkForFolders: false,
            randomNames: false,
            useCopy: false,
            toolBar: true,

            files: [],
            folders: [],
            directories: [],
            filterdList: [],
            bulkList: [],
            lockedList: [],

            moveToPath: null,
            selectedFile: null,
            currentFileIndex: null,
            sortBy: null,
            currentFilterName: null,
            searchItemsCount: null,
            searchFor: null,
            new_folder_name: null,
            new_filename: null,
            active_modal: null,

            navDirection: null,
            gradients: [
                'linear-gradient(141deg, #009e6c 0%, #00d1b2 71%, #00e7eb 100%)',
                'linear-gradient(141deg, #04a6d7 0%, #209cee 71%, #3287f5 100%)',
                'linear-gradient(141deg, #12af2f 0%, #23d160 71%, #2ce28a 100%)',
                'linear-gradient(141deg, #ffaf24 0%, #ffdd57 71%, #fffa70 100%)',
                'linear-gradient(141deg, #ff0561 0%, #ff3860 71%, #ff5257 100%)',
                'linear-gradient(141deg, #1f191a 0%, #363636 71%, #46403f 100%)'
            ]
        }
    },
    created() {
        this.preSaved()

        if (this.checkForRestrictedPath()) {
            return this.restrictAccess()
        }

        this.getFiles(this.folders, null, this.selectedFile)

        document.addEventListener('keydown', this.shortCuts)
    },
    mounted() {
        this.fileUpload()
    },
    updated() {
        this.autoPlay()
        this.$nextTick(() => {
            return this.$refs.move_folder_dropdown.options[0]
                ? this.checkForFolders = true
                : this.checkForFolders = false
        })
    },
    beforeDestroy() {
        document.removeEventListener('keydown', this.shortCuts)
    },
    methods: {
        preSaved() {
            let ls = this.$ls.get('mediamanager')

            if (ls) {
                this.randomNames = ls.randomNames == 'undefined' ? false : ls.randomNames
                this.folders = ls.folders == 'undefined' ? [] : ls.folders
                this.toolBar = ls.toolBar == 'undefined' ? true : ls.toolBar
                this.selectedFile = ls.selectedFileName == 'undefined' ? null : ls.selectedFileName
            }
        },

        fileUpload() {
            const manager = this

            let items = 0
            let progress = 0
            let counter = 0
            let last = null
            let sendingComplete = false

            new Dropzone('#new-upload', {
                createImageThumbnails: false,
                parallelUploads: 10,
                hiddenInputContainer: '#new-upload',
                uploadMultiple: true,
                forceFallback: false,
                ignoreHiddenFiles: true,
                timeout: 3600000,
                previewsContainer: '#uploadPreview',
                addedfile() {
                    manager.showProgress = true
                    items++
                    counter = 100 / items
                },
                sending() {
                    progress += counter
                    manager.progressCounter = `${progress.toFixed(2)}%`

                    if (progress >= 100) {
                        sendingComplete = true
                    }
                },
                successmultiple(files, res) {
                    res.data.map((item) => {
                        if (item.success) {
                            manager.showNotif(`${manager.trans('upload_success')} "${item.message}"`)
                            last = item.message
                        } else {
                            manager.showNotif(item.message, 'danger')
                        }
                    })

                    if (sendingComplete) {
                        items = 0
                        progress = 0
                        manager.progressCounter = 0
                        manager.showProgress = false

                        manager.$refs['success-audio'].play()

                        last
                            ? manager.getFiles(manager.folders, null, last)
                            : manager.getFiles(manager.folders)
                    }
                },
                errormultiple(files, res) {
                    manager.showNotif(res, 'danger')
                }
            })
        },
        /* end of upload */

        shortCuts(e) {
            if (!(this.isLoading || e.altKey || e.ctrlKey || e.metaKey)) {
                // when modal isnt visible
                if (!this.active_modal) {
                    // when search is not focused
                    if (document.activeElement.dataset.search == undefined) {
                        // when no bulk selecting
                        if (!this.isBulkSelecting()) {

                            // open folder
                            if (keycode(e) == 'enter' && this.selectedFile) {
                                this.openFolder(this.selectedFile)
                            }

                            // go up a dir
                            if (keycode(e) == 'backspace' && this.folders.length) {
                                this.goToPrevFolder()
                            }

                            // when there are files
                            if (this.allItemsCount) {
                                this.navigation(e)

                                if (
                                    keycode(e) == 'space' &&
                                    e.target == document.body &&
                                    (
                                        this.selectedFileIs('video') ||
                                        this.selectedFileIs('audio') ||
                                        this.selectedFileIs('image') ||
                                        this.selectedFileIs('pdf') ||
                                        this.selectedFileIs('text')
                                    )
                                ) {
                                    e.preventDefault()

                                    // play-pause media
                                    if (this.selectedFileIs('video') || this.selectedFileIs('audio')) {
                                        let player = this.$refs.player

                                        return player.paused
                                            ? player.play()
                                            : player.pause()
                                    }

                                    // "show" image/pdf/text quick view
                                    if (this.selectedFileIs('image') || this.selectedFileIs('pdf') || this.selectedFileIs('text')) {
                                        this.noScroll('add')
                                        this.toggleModal('preview_modal')
                                    }
                                }
                            }
                            // end of when there are files

                            // refresh
                            if (keycode(e) == 'r') {
                                this.refresh()
                            }

                            // file upload
                            if (keycode(e) == 'u') {
                                this.$refs.upload.click()
                            }
                        }
                        /* end of no bulk selection */

                        // with or without bulk selection
                        if (this.allItemsCount) {
                            // bulk select
                            if (keycode(e) == 'b') {
                                if (this.searchFor && this.searchItemsCount == 0) {
                                    return
                                }

                                if (this.$refs.bulkSelect) {
                                    this.$refs.bulkSelect.click()
                                }
                            }

                            if (this.isBulkSelecting()) {
                                // add all to bulk list
                                if (keycode(e) == 'a') {
                                    if (this.$refs.bulkSelectAll) {
                                        this.$refs.bulkSelectAll.click()
                                    }
                                }

                                // cancel bulk selection
                                if (keycode(e) == 'esc') {
                                    if (this.$refs.bulkSelect) {
                                        this.$refs.bulkSelect.click()
                                    }
                                }
                            }

                            // delete file
                            if (keycode(e) == 'delete' || keycode(e) == 'd') {
                                if (!this.$refs.delete[0].disabled) {
                                    this.$refs.delete[0].click()
                                }
                            }

                            // move file
                            if (this.checkForFolders && keycode(e) == 'm') {
                                if (!this.$refs.move[0].disabled) {
                                    this.$refs.move[0].click()
                                }
                            }

                            // lock files
                            if (keycode(e) == 'l') {
                                if (this.$refs.lock) {
                                    this.$refs.lock.click()
                                }
                            }
                        }
                        /* end of with or without bulk selection */

                        // toggle file details sidebar
                        if (keycode(e) == 't') {
                            this.toggleInfoPanel()
                        }
                    }
                    /* end of search is not focused */

                    // cancel search
                    else if (keycode(e) == 'esc') {
                        this.resetInput('searchFor')
                    }
                }
                /* end of modal isnt visible */

                // when modal is visible
                else {
                    if (this.isActiveModal('preview_modal')) {
                        // hide lb
                        if (keycode(e) == 'space') {
                            e.preventDefault()
                            this.toggleModal()
                        }

                        this.navigation(e)
                    }

                    // hide lb
                    if (keycode(e) == 'esc') {
                        this.toggleModal()
                    }
                }
                /* end of modal is visible */
            }
        },
        /* end of short cuts */

        refresh() {
            this.getFiles(this.folders, null, this.selectedFile.name)
        },
        deleteItem() {
            if (this.$refs.delete[0].disabled) {
                return
            }

            if (!this.isBulkSelecting() && this.selectedFile) {
                this.selectedFileIs('folder')
                    ? this.folderWarning = true
                    : this.folderWarning = false
            }

            if (this.bulkItemsCount) {
                this.bulkListFilter.some((item) => {
                    return this.fileTypeIs(item, 'folder')
                        ? this.folderWarning = true
                        : this.folderWarning = false
                })
            }

            this.toggleModal('confirm_delete_modal')
        },
        moveItem() {
            if (this.$refs.move[0].disabled) {
                return
            }

            this.toggleModal('move_file_modal')
        },
        renameItem() {
            this.toggleModal('rename_file_modal')
        },
        blkSlct() {
            this.bulkSelect = !this.bulkSelect
            this.bulkSelectAll = false
            this.resetInput('bulkList', [])
            this.resetInput(['selectedFile', 'currentFileIndex'])

            if (!this.isBulkSelecting()) {
                this.selectFirst()
            }
        },
        blkSlctAll() {
            // if no items in bulk list
            if (this.bulkList == 0) {
                // if no search query
                if (!this.searchFor) {
                    this.bulkSelectAll = true
                    this.bulkList = this.allFiles.slice(0)
                }

                // if found search items
                if (this.searchFor && this.searchItemsCount) {
                    this.bulkSelectAll = true

                    let list = this.filesList
                    for (let i = list.length - 1; i >= 0; i--) {
                        list[i].click()
                    }
                }
            }

            // if having search + having bulk items < search found items
            else if (this.searchFor && this.bulkItemsCount < this.searchItemsCount) {
                this.resetInput('bulkList', [])
                this.resetInput(['selectedFile', 'currentFileIndex'])

                if (this.bulkSelectAll) {
                    this.bulkSelectAll = false
                } else {
                    this.bulkSelectAll = true

                    let list = this.filesList
                    for (let i = list.length - 1; i >= 0; i--) {
                        list[i].click()
                    }
                }
            }

            // if NO search + having bulk items < all items
            else if (!this.searchFor && this.bulkItemsCount < this.allItemsCount) {
                if (this.bulkSelectAll) {
                    this.bulkSelectAll = false
                    this.resetInput('bulkList', [])
                } else {
                    this.bulkSelectAll = true
                    this.bulkList = this.allFiles.slice(0)
                }

                this.resetInput(['selectedFile', 'currentFileIndex'])
            }

            // otherwise
            else {
                this.bulkSelectAll = false
                this.resetInput('bulkList', [])
                this.resetInput(['selectedFile', 'currentFileIndex'])
            }

            // if we have items in bulk list, select first item
            if (this.bulkItemsCount) {
                this.selectedFile = this.bulkList[0]
            }
        },

        /**
         * autoplay media
         */
        autoPlay() {
            if (this.filterNameIs('audio') || this.filterNameIs('video')) {
                let player = this.$refs.player
                if (player) {
                    player.onended = () => {
                        // stop at the end of list
                        if (this.currentFileIndex < this.allItemsCount - 1) {
                            // nav to next
                            this.goToNext()

                            // play navigated to
                            this.$nextTick(() => {
                                this.$refs.player.play()
                            })
                        }
                    }
                }
            }
        },

        /**
         * animation
         *
         * because $nextTick doesnt work correctly
         * with transition-group
         */
        afterEnter() {
            if (this.searchFor) {
                this.updateSearchCount()
            }
        },
        afterLeave() {
            if (this.searchFor) {
                this.updateSearchCount()
            }

            if (!this.allItemsCount) {
                this.resetInput(['selectedFile', 'currentFileIndex'])
            }
        }
    },
    render() {}
}
</script>
