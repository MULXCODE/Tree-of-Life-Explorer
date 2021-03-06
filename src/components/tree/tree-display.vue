<template lang="pug">
.tree
  //- ChildMenu(:leaf="subtreeLeaf", :x="childMenuX", :y="childMenuY", @select="$emit('add-node', arguments[0])")
  //- .lvl(v-for="level in columns")
  //-   .col(v-for="col in level")
  //-     template(v-if="!col")
  //-       .spacer
  //-     .tol-col(v-if="col")
  //-       .box {{ col.tree.lineage.length }}
  //-       TOLNodeCard(
  //-         v-if="col.tree.leaf"
  //-         , :leaf="col.tree.leaf"
  //-       )
  template(v-if="compactView")
    .tol-leaf(
      v-for="branch in branches"
      , :class="{ 'horizontal': horizontal }"
      , :style="{ left: `${branch.x + branch.dx}px`, top: `${branch.y + branch.dy}px` }"
      , :tabindex="-1"
      )
      Connection(v-if="!(branch.isRoot && !branch.tree.lineage.length)", :horizontal="horizontal", :x1="branch.px", :y1="branch.py", :x2="branch.x + branch.dx", :y2="branch.y + branch.dy", :padding="branch.dx || branch.dy")
      CondensedNode(v-if="!branch.tree.split.length", :leaf="branch.tree.leaf", :x="branch.x + branch.dx", :y="branch.y", :horizontal="horizontal", @click="$emit('zoom-in', arguments[0])")

  transition-group(name="tree", v-if="!compactView", :css="transitions")
    .tol-leaf(
      v-for="branch in branches"
      , :class="{ 'horizontal': horizontal }"
      , :key="branch.key"
      , :style="{ left: `${branch.x-(horizontal ? 0 : 0.5 * cardWidth) + branch.dx}px`, top: `${branch.y-(horizontal ? 0.5 * cardHeight : 0) + branch.dy}px`, width: cardWidth + 'px', height: cardHeight + 'px' }"
      , :tabindex="-1"
      , :data-ott="branch.tree.leaf.node_id"
      )
      Connection(v-if="!(branch.isRoot && !branch.tree.lineage.length)", :horizontal="horizontal", :x1="branch.px", :y1="branch.py", :x2="branch.x + branch.dx", :y2="branch.y + branch.dy", :padding="(horizontal ? cardWidth : cardHeight) * 1.7")
      //- Node(v-if="branch.tree.lineage.length", :tree="branch.tree", :x="branch.x", :y="branch.y", @click="$emit('leaf-click', arguments[0])")
      AncestorCollapse(v-if="branch.tree.lineage.length", :tree="branch.tree", :x="branch.x", :y="branch.y", :horizontal="horizontal", @click="$emit('leaf-click', arguments[0])")

      TOLLeafView(
        :leaf="branch.tree.leaf"
        , :truncate-length="truncateLength"
        , :hide-images="hideImages"
        , :flap-style="flapStyle"
        , :horizontal="horizontal"
        , @remove="$emit( 'remove', branch.tree.leaf )"
        , @cut="$emit( 'cut', branch.tree )"
        , @error="$emit( 'error', arguments[0] )"
        , @add-node="$emit( 'add-node', arguments[0] )"
        , @remove-node="$emit( 'remove', arguments[0] )"
        )
</template>

<script>
import TOLLeafView from '@/components/tol-leaf-view'
import TOLMRCAView from '@/components/tol-mrca-view'
import Node from './node'
import AncestorCollapse from './ancestor-collapse'
import CondensedNode from './condensed-node'
import Tail from './tail'
import Connection from './connection'
import ChildMenu from './child-menu'
import _flatten from 'lodash/flatten'

function getBranches( tree, opts, x = 0, y = 0, level = 0 ){

  let hz = opts.horizontal
  let height = opts.cardHeight
  let nodeContainerSize = 0.5 * (hz ? height : opts.cardWidth) + opts.padding
  let count = tree.nTips
  let colstart = -(count - 1)
  let branchSpacing = opts.branchSpacing
  let branches = []

  branches.push({
    tree
    , x
    , y
    , px: opts.px === undefined ? x : opts.px
    , py: opts.py === undefined ? y : opts.py
    , key: (tree.leaf.node_id || tree.leaf)
    , dx: hz ? nodeContainerSize : 0
    , dy: hz ? 0 : 0.75 * height
    , isRoot: level === 0
    , hasSplit: !!tree.split.length
  })

  if ( !tree.split.length ){
    return branches
  }

  branches = branches.concat(
    _flatten(
      tree.split.map( (subtree, idx) => {
        let col = subtree.nTips - 1
        var xpos, ypos

        if ( hz ){
          xpos = x + branchSpacing + 1.3 * opts.cardWidth
          ypos = y + (col + colstart) * nodeContainerSize
        } else {
          xpos = x + (col + colstart) * nodeContainerSize
          ypos = y + branchSpacing + 1.3 * height
        }

        colstart = 2 * col + colstart + 2

        return getBranches( subtree, {...opts, px: x, py: y}, xpos, ypos, level + 1 )
      })
    )
  )

  return branches
}

function appendToColumns( tree, columns, colOffset = 0, level = 0 ){
  let colIndex = colOffset + tree.nTips - 1
  let row = columns[ level ]
  row[ colIndex ] = {
    tree
    , key: (tree.leaf ? tree.leaf.node_id : tree.lineage[0].node_id + level)
    , isRoot: level === 0
    , hasSplit: !!tree.split
  }

  if ( !tree.split ){
    return columns
  }

  tree.split.forEach( (subtree, idx) => {
    appendToColumns( subtree, columns, colOffset, level + 1 )
    colOffset += subtree.nTips * 2
  })

  return columns
}

function getColumns( tree ){
  // const nCols = tree.nTips * 2 - 1
  const columns = Array( tree.depth ).fill(0).map(a => [])
  appendToColumns( tree, columns )
  return columns
}

export default {
  name: 'Tree'
  , props: {
    'tree': Object
    , 'x': Number
    , 'y': Number
    , 'px': Number
    , 'py': Number
    , 'cardWidth': Number
    , 'cardHeight': Number
    , 'padding': Number
    , 'branchSpacing': Number
    , 'horizontal': Boolean
    , 'flapStyle': Boolean
    , 'hide-images': Boolean
    , 'computedBranches': null
    , 'compactView': Boolean
    , 'transitions': {
      type: Boolean
      , default: true
    }
  }
  , data: () => ({
    subtreeLeaf: null
    , childMenuX: 0
    , childMenuY: 0
  })
  , components: {
    Connection
    , ChildMenu
    , Node
    , AncestorCollapse
    , CondensedNode
    , Tail
    , TOLLeafView
    , TOLMRCAView
  }
  , created () {
    document.addEventListener('click', this.onDocumentClick)
  }
  , destroyed () {
    // important to clean up!!
    document.removeEventListener('click', this.onDocumentClick)
  }
  , computed: {
    branches(){
      let tree = this.tree
      let x = this.x
      let y = this.y
      let b = getBranches(tree, {
        cardWidth: this.compactView ? 0 : this.cardWidth
        , cardHeight: this.compactView ? 0 : this.cardHeight
        , branchSpacing: this.compactView ? 40 : this.branchSpacing
        , padding: this.compactView ? 10 : this.padding
        , horizontal: this.horizontal
      }, x, y)
      this.$emit('update:computedBranches', b)
      return b
    }

    , columns(){
      let tree = this.tree
      let cols = getColumns( tree )
      return cols
    }

    , truncateLength(){
      return this.cardWidth / 9 | 0
    }
  }
  , methods: {
    openChildMenu({ x, y, leaf }){
      this.childMenuX = x
      this.childMenuY = y
      this.subtreeLeaf = leaf
    }
    , onDocumentClick(){
      this.subtreeLeaf = null
    }
  }
}
</script>

<style lang="scss" scoped>
@import '@/styles/_variables.scss';
.child-menu {
  position: absolute;
  top: 0;
  left: 0;
  .dropdown-menu {
    position: relative;
    top: 20px;
    left: -50%;
  }
}
.lvl {
  display: flex;
  align-items: flex-start;
  .col {
    display: block;
    margin: 0 -60px;
    width: 260px;

    &:first-child {
      margin-left: 0;
    }

    &:last-child {
      margin-right: 0;
    }
  }

  .spacer {
    height: 100px;
  }
}
.tol-leaf {
  position: absolute;
  top: 0;
  left: 0;
  outline: none;

  &.horizontal {
    display: flex;
    flex-direction: column;
    justify-content: center;
  }

  &:hover,
  &:active,
  &:focus {
    z-index: 1;
  }

  .card {
    position: absolute;
    top: 0;
    left: 0;
  }
}
.tree-move {
  transition: all 0.7s ease-in-out;
}
.tree-enter-active, .tree-leave-active {
  transition: all 0.7s ease-in-out;
}
.tree-enter, .tree-leave-to {
  transition: all 0.7s ease-in-out;
  opacity: 0;
  top: -80px;
}
.simple {
  background: $white;
}
</style>
